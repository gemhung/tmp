# What changes would you suggest in a code review? Would you check in your code like this? Do you think the code is clean? 
1. The `getCount` is called without `protection` by lock and it's something weird to me
```c++
    uint64_t getCount() const {
        return mWritePtr > mReadPtr ? mWritePtr - mReadPtr : mReadPtr - mWritePtr;
    }
```
2. I'm not sure if it's possible when `mWrite < mReadPtr` cause both are increasing and never wrap around ? What's more, I try running with simply `mWritePtr-mReadPtr` implementation and it can run for a long time without error
```c++
    uint64_t getCount() const {
        // return mWritePtr > mReadPtr ? mWritePtr - mReadPtr : mReadPtr - mWritePtr;
        retur nWritePtr - mReadPtr; // this can run for a long time without error
    }
```

1. Move the following two methonds out of `Concurrentqueue` class cause they're independent from concurrentqueue itself 
```c++
    static constexpr unsigned Log2(unsigned n, unsigned p = 0) {
        return (n <= 1) ? p : Log2(n / 2, p + 1);
    }

    static constexpr uint64_t closestExponentOf2(uint64_t x) {
        return (1UL << ((uint64_t) (Log2(SIZE - 1)) + 1));
    }
```
2. Return refernece in this case isn't a good idea cause it might be overwritten by other proudcuer and later cause double free
```c++
    //const T& pop() {
    T pop() {
        ...
    }
```

3. There are many ways to implemnet concurrentqueue. A possible way is to use `conditional variable` + `queue` to simplfy the design. 
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
4. `mWritePtr` and `mReadPtr` might overflow
```c++
        mWritePtr++;
        mReadPtr++;
```
5. Throwing `queue is full` runtime_error in production might not be a good idea ? Often we'll wait for space rather than throwing exception  
```c++
        if (!busyWaitForPush()) {
            throw std::runtime_error("Concurrent queue full cannot write to it!");
        }
```

6. Better to disable copy constructor and copy assingmen if we're not using them
```
// Suggestion
	ConcurrentQueue(ConcurrentQueue const&) = delete;
	ConcurrentQueue& operator=(ConcurrentQueue const&) = delete;
```

7. We can return `std::optional` rather than `mEmpty` to simply the design
```c++
    const T& pop() {
        if (!peek()) {
            return mEmpty;
        }
        ...
    }
```

8. Using new / delete in modern c++ can cause a lot of problems. We can use `shared_ptr`/`unique_ptr` to gain more safty 
```c++
 delete task;
 ...
 auto newTask = new Functor([ = ] {
```

9. What if `T` didin't support default consturctor ?
```c++
template<typename T, uint64_t SIZE, uint64_t MAX_SPIN_ON_BUSY>
const T ConcurrentQueue<T, SIZE, MAX_SPIN_ON_BUSY>::mEmpty = T{ };
```
# Do you see room for improving the performance without breaking thread safety? 
* I'm not sure whether this 2 pointers can really work on the production env cause it wasn't common for multi-producer / multi-consumers env. For now, I can't leave comment about performance.
* On the other hand, for the big picture, 
    * One thing we can try is to implement it with lock-free algo rather than mutex to see if we can gain performance
    * If it's intended to run on certain platform. Perhaps we can implmenet it based on the platform (Unix, Windows or MACH)

# What steps would you take to find the bug?
* Try to reproduce the bug 
* Turn off optimization 
* Compile and run wiht `-fsanitize=address` and it shows with `heap-use-after-free` error
* The `pop` method is suspecious cause it's return a reference which is not a good idea in multi-thread env. I try to let it return a simple `T`
* Cannot reproduce the bug    
    
# If you can find the bug, what fix would you suggest?
* For now, I can reproduce the segament fault bug immediatly. In my exp, returning a reference in multi-thread env isn't often a good idea. My suggestion is to return a `T` and let compiler do the optimization of `RVO`/`NRVO`. 
* The fix is to return a `T` rather than a `const T&` and it could run for long time with this fix
```c++
// Suggestion return a T
    //const T& pop() {
    T pop() {
        ...
    }
```
 # Write a proper test case to prove that the bug is solved (may be in pseudo code), preferably in a deterministic matter?
 * I cannot find a deterministic matter to reproduce this bug. 


# If you cannot find the bug, how would you go about testing the thread safety of the class? 
* Testing multi-thread is hard. Harder than we thought. I'd like to start by referencing some open source projects to find out how they were tested in multi-thread env. 
* Here is a popular open source project of `concurrentqueue`. https://github.com/cameron314/concurrentqueue. 
* We might able to learn from how it's tested https://github.com/cameron314/concurrentqueue/tree/master/tests
* Run with `-fsanitize=address`
* Test with 3rd party tool such `Valgrind`

    
