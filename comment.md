# What changes would you suggest in a code review? Would you check in your code like this? Do you think the code is clean? 
1. The `getCount` is used without `protection` by lock and it's something weird to me
```c++
    uint64_t getCount() const {
        return mWritePtr > mReadPtr ? mWritePtr - mReadPtr : mReadPtr - mWritePtr;
    }
```
2. Move the following two methonds out of Concurrentqueue cause they're independent from concurrentqueue itself 
```c++
    static constexpr unsigned Log2(unsigned n, unsigned p = 0) {
        return (n <= 1) ? p : Log2(n / 2, p + 1);
    }

    static constexpr uint64_t closestExponentOf2(uint64_t x) {
        return (1UL << ((uint64_t) (Log2(SIZE - 1)) + 1));
    }
```
3. Return refernece in this case isn't a good idea cause it might be overwritten by other proudcuer and later cause double free
```c++
    //const T& pop() {
    T pop() {
        ...
    }
```

4. There are many ways to implemnet concurrentqueue. A possible way is to use `conditional variable` + `queue` to simplfy the design. 
```c++
/*
    void push(const T& pItem) {
        if (!busyWaitForPush()) {
            throw std::runtime_error("Concurrent queue full cannot write to it!");
        }

        std::lock_guard<std::mutex> lock(mLock);
        mMem[mWritePtr & mRingModMask] = pItem;
        mWritePtr++;
    }
 */
 // Suggestion
 	template <typename... Args, typename Rep, typename Period>
	bool tryPush(const std::chrono::duration<Rep, Period>& timeout, Args&&... args) {
		std::unique_lock<std::mutex> lk(m_mtx);
		if (isFull()) {
			if (!m_condition_pop.wait_for(lk, timeout, [this] { return !isFull(); }))
				return false;
		}
		m_task_queue.emplace(std::forward<Args>(args)...);
		m_condition_push.notify_one();
		return true;
	}
```
5. `mWritePtr` and `mReadPtr` might overflow
```c++
        mWritePtr++;
        mReadPtr++;
```
6. The if throwing runtime_error in production might not be a good idea ?  
```c++
        if (!busyWaitForPush()) {
            throw std::runtime_error("Concurrent queue full cannot write to it!");
        }
```

7. Better to disable copy constructor and copy assingmen if we're not using them
```
// Suggestion
	ConcurrentQueue(ConcurrentQueue const&) = delete;
	ConcurrentQueue& operator=(ConcurrentQueue const&) = delete;
```

7. We can return `std::optional` rather than `mEmpty` to simply the design 
```c++
        if (!peek()) {
            return mEmpty;
        }
```
# Do you see room for improving the performance without breaking thread safety? 
* I'm not sure whether this 2 pointers can really work on the production cause it wan't common for multi-producer / multi-consumers. Hence I don't wanna leave comment for now.
* To the big picture,
    * One thing we can try is to implement it with lock-free algo rather than mutex to see if we can gain performance
    * If it's intended to run on certain platform. Perhaps we can implmenet it based on the platform (Unix, Windows or MACH). 
    * Chances are accessing platform specific api(Unic, Windows or MAC) might be faster than `c++` generic impementation

# What steps would you take to find the bug? 
    a. Narrow down the code size 
    b. Isolate the class use case and run it with a minimum implementation to see if can reproduce the bug
    c. Running more edge cases to see if it can be reproduced
    d. Print out suspicious variables values whenever it's possible
    e. Disblae optimization to see if we can reproduce the bug
    f. If it's obvious that the bug is some kind of concurrency problem, it's likely to happen where data was exchanged/called/assgined like `pop` or `push`. We can review these peice of code.
    g. Try not to use `std::move()` or `return reference / pointer ` by making things work slowly with copied values to see if we can repdouce the bug cause it might be tricky to use those in multi-thread env
    
# If you can find the bug, what fix would you suggest? Write a proper test case to prove that the bug is solved (may be in pseudo code), preferably in a deterministic matter?
* For now, I'd say returning a reference in multi-thread env isn't a good idea in my experience. My suggestion is to return a `T` and let compiler do the optimization of `RVO`/`NRVO`
```c++
// Suggestion return a T
    //const T& pop() {
    T pop() {
        ...
    }
```

# If you cannot find the bug, how would you go about testing the thread safety of the class? 
* Testing multi-thread is hard. Harder than we thought. I'd like to start by referencing some open source projects to find out how they were tested in multi-thread env. 
* Here is a popular open source project of `concurrentqueue`. https://github.com/cameron314/concurrentqueue. 
* We might able to learn from how it's tested https://github.com/cameron314/concurrentqueue/tree/master/tests
