5 Tips for Unit Testing Threaded Code
===
Here's a few tips on how take make testing your code for logical correctness (as opposed to multi-threaded correctness).

I find that there are essentially two stereotypical patterns with threaded code:

1. Task orientated -  many, short running, homogeneous tasks, often run within the Java 5 executor framework,
1. Process orientated - few, long running, heterogeneous tasks, often event based (waiting on notification), or polling (sleeping between cycles), often expressed using a thread or runnable.

Testing either type of code can be hard; the work is done in another thread, and therefore notification of completion can be opaque, or is hidden behind a level of abstraction.

The code is [on GitHub](https://github.com/alexec/threaded-code-testing).

Tip 1 - Life-cycle Manage Your Objects
---

Object that have a managed life-cycle are are easier to test, the life-cycle allows for set-up and tear-down, which means you can clean-up after your test and no spurious threads are lying around to pollute other tests.

    public class Foo {
        private ExecutorService executorService;

        public void start() {
            executorService = Executors.newSingleThreadExecutor();
        }

        public void stop() {
            executorService.shutdown();
        }
    }

Tip 2 - Set a Timeout on Your Tests
---

Bugs in code (as you'll see below) can result in a multi-threaded test never completing, as (for example) you're waiting on some flag that never gets set. JUnit lets you set a timeout on your test.

    ...
    @Test(timeout = 100) // in case we never get a notification
    public void testGivenNewFooWhenIncrThenGetOne() throws Exception {
    ...

Tip 3 - Run Tasks in the Same Thread as Your Test
---

Typically you'll have an object that runs tasks in a thread pool. This means that your unit test might have to wait for the task to complete, but you're not able to know when it would complete. You might guess, for example:

    public class Foo {
        private final AtomicLong foo = new AtomicLong();
    ...
        public void incr() {
            executorService.submit(new Runnable() {
                @Override
                public void run() {
                    foo.incrementAndGet();
                }
            });
        }
    ...
        public long get() {
            return foo.get();
        }
    }

Consider this test:

    public class FooTest {

        private Foo sut; // system under test

        @Before
        public void setUp() throws Exception {
            sut = new Foo();
            sut.start();
        }

        @After
        public void tearDown() throws Exception {
            sut.stop();
        }

        @Test
        public void testGivenFooWhenIncrementGetOne() throws Exception {
            sut.incr();
            Thread.sleep(1000); // yuk - a slow test - don't do this
            assertEquals("foo", 1, sut.get());
        }
    }

But this is problematic. Execution is non-uniform so there's no guarantee that this will work on another machine. It's fragile, changes to the code can cause the test to fail as it suddenly take a bit too long. Its slow, as you will be generous with sleep when it fails.

A trick is to make the task run synchronously, i.e. in the same thread as the test. Here this can be achieved by injecting the executor:

    public class Foo {
    ...
        public Foo(ExecutorService executorService) {
            this.executorService = executorService;
        }
    ...
        public void stop() {
            // nop
        }

Then you can have use a synchronous executor service (similar in concept to a SynchronousQueue) to test:

    public class SynchronousExecutorService extends AbstractExecutorService {
        private boolean shutdown;

        @Override
        public void shutdown() {shutdown = true;}

        @Override
        public List&lt;Runnable&gt; shutdownNow() {shutdown = true; return Collections.emptyList();}

        @Override
        public boolean isShutdown() {shutdown = true; return shutdown;}

        @Override
        public boolean isTerminated() {return shutdown;}

        @Override
        public boolean awaitTermination(final long timeout, final TimeUnit unit) {return true;}

        @Override
        public void execute(final Runnable command) {command.run();}
    }

An updated test that doesn't need to sleep:

    public class FooTest {

        private Foo sut; // system under test
        private ExecutorService executorService;

        @Before
        public void setUp() throws Exception {
            executorService = new SynchronousExecutorService();
            sut = new Foo(executorService);
            sut.start();
        }

        @After
        public void tearDown() throws Exception {
            sut.stop();
            executorService.shutdown();
        }

        @Test
        public void testGivenFooWhenIncrementGetOne() throws Exception {
            sut.incr();
            assertEquals("foo", 1, sut.get());
        }
    }

Note that you need to life-cycle manage the executor externally to Foo.

Tip 4 - Extract the Work from the Threading
---

If your thread is waiting for an event, or a time before it does any work, extract the work to its own method and call it directly. Consider this:

    public class FooThread extends Thread {
        private final Object ready = new Object();
        private volatile boolean cancelled;
        private final AtomicLong foo = new AtomicLong();

        @Override
        public void run() {
            try {
                synchronized (ready) {
                    while (!cancelled) {
                        ready.wait();
                        foo.incrementAndGet();
                    }
                }
            } catch (InterruptedException e) {
                e.printStackTrace(); // bad practise generally, but good enough for this example
            }
        }

        public void incr() {
            synchronized (ready) {
                ready.notifyAll();
            }
        }

        public long get() {
            return foo.get();
        }

        public void cancel() throws InterruptedException {
            cancelled = true;
            synchronized (ready) {
                ready.notifyAll();
            }
        }
    }

And this test:

    public class FooThreadTest {

        private FooThread sut;

        @Before
        public void setUp() throws Exception {
            sut = new FooThread();
            sut.start();
            Thread.sleep(1000); // yuk
            assertEquals("thread state", Thread.State.WAITING, sut.getState());
        }

        @After
        public void tearDown() throws Exception {
            sut.cancel();
        }

        @After
        public void tearDown() throws Exception {
            sut.cancel();
        }

        @Test
        public void testGivenNewFooWhenIncrThenGetOne() throws Exception {
            sut.incr();
            Thread.sleep(1000); // yuk
            assertEquals("foo", 1, sut.get());
        }
    }

Now extract the work:

	@Override
	public void run() {
		try {
			synchronized (ready) {
				while (!cancelled) {
					ready.wait();
					undertakeWork();
				}
			}
		} catch (InterruptedException e) {
			e.printStackTrace(); // bad practise generally, but good enough for this example
		}
	}

	void undertakeWork() {
		foo.incrementAndGet();
	}

Re-factor the test:

    public class FooThreadTest {

        private FooThread sut;

        @Before
        public void setUp() throws Exception {
            sut = new FooThread();
        }

        @Test
        public void testGivenNewFooWhenIncrThenGetOne() throws Exception {
            sut.incr();
            sut.undertakeWork();
            assertEquals("foo", 1, sut.get());
        }
    }


Tip 5 - Notify State Change via Events
---

An alternative to the previous two tips is to use a notification system, so your test can listen to the threaded object.

Here's a task oriented example:

    public class ObservableFoo extends Observable {
        private final AtomicLong foo = new AtomicLong();
        private ExecutorService executorService;
    
        public void start() {
            executorService = Executors.newSingleThreadExecutor();
        }
    
        public void stop() {
            executorService.shutdown();
        }
    
        public void incr() {
            executorService.submit(new Runnable() {
                @Override
                public void run() {
                    foo.incrementAndGet();
                    setChanged();
                    notifyObservers(); // lazy use of observable
                }
            });
        }
    
        public long get() {
            return foo.get();
        }
    }

And its corresponding test (note the use of timeout):

    public class ObservableFooTest implements Observer {
    
        private ObservableFoo sut;
        private CountDownLatch updateLatch; // used to react to event
    
        @Before
        public void setUp() throws Exception {
            updateLatch = new CountDownLatch(1);
            sut = new ObservableFoo();
            sut.addObserver(this);
            sut.start();
        }
    
        @Override
        public void update(final Observable o, final Object arg) {
            assert o == sut;
            updateLatch.countDown();
        }
    
        @After
        public void tearDown() throws Exception {
            sut.deleteObserver(this);
            sut.stop();
        }
    
        @Test(timeout = 100) // in case we never get a notification
        public void testGivenNewFooWhenIncrThenGetOne() throws Exception {
            sut.incr();
            updateLatch.await();
            assertEquals("foo", 1, sut.get());
        }
    }

This has pros and cons:

Pros:

1. Creates useful code for listening to the object.
1. Can take advantage of existing notification code, which makes it a good choice where that already exists.
1. Is more flexible, can apply to both tasks and process orientated code.
1. It is more cohesive than extracting the work.

Cons:

1. Listener code can be complex and introduce its own problems, creating additional production code that ought to be tested.
1. De-couples submission from notification.
1. Requires you to deal with the scenario that no notification is sent (e.g. due to bug).
1. Test code can be quite verbose and therefore prone to having bugs.
