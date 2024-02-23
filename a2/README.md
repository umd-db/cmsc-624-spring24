## Assignment 2: Concurrency Control Schemes

In this assignment you will be implementing **six** concurrency control schemes:

- Two versions of locking schemes, both of which implement variations of the standard two-phase locking algorithm we discussed in class.
- A version of OCC very similar to the serial-validation version described in the OCC paper you read for class.
- A version of OCC somewhat similar to the parallel-validation version described in the OCC paper.
- A version of MVCC timestamp ordering that is a simplified version of the PostgreSQL scheme we studied in class.
- A version of Serializable version of MVCC timestamp ordering that is a simplified version of PostgreSQL's Serializable Snapshot Isolation (SSI) scheme.

- Assignment 2:  The latex template for your submission can be found in cmsc624-a2-sol-template. You can upload the template to [Overleaf](https://www.overleaf.com/).


Similar to assignment 1, aside from writing code, this assignment involves **running some performance experiments** and **answering some questions** we ask below. We use a private git repository for each student, and each student can commit changes and push them to this private repository. Our grading server will run your code regularly, generate the output files, and commit them back to you. This will enable the code from each student to run on an identical server (with the same performance characteristics), so that students do not have to be concerned that any strange performance numbers they may see is the result of their particular choice of hardware. 

### Framework

You'll be implementing these concurrency control schemes within a transaction processing framework that implements a simple, main-memory resident key-value store. This is a prototype system designed specially for this assignment.

In the code, you'll see that it contains two subdirectories ---`txn` and `utils`. Nearly all of the source files you need to worry about are in the `txn` subdirectory, though you might need to occasionally take a peek at a file or two in the `util` subdirectory.

To build and test the system, you can run

```shell
mkdir build && cmake ..
make -j8
ctest -V
```

At any time, this will first compile the system; if this succeeds with no errors, it will also run two test scripts: one which performs some basic correctness tests of your lock manager implementation, and a second which profiles performance of the system. This second one takes a number of minutes to run, but you can cancel it at any time by pressing `ctrl-C`. 

> You can not pass the lock manager test right now until you complete the lock manager implementation.

When implementing your solutions, please:

- Comment your header files & code thoroughly in the manner demonstrated in the rest of the framework.
- Organize your code logically.
- Use descriptive variable names.

### Codebase

In this assignment, you will need to make changes to the following files/classes/methods:

* txn/lock_manager.cc: All methods for classes `LockManagerA` (Part 1A) and `LockManagerB` (Part 1B).
* txn/txn_processor.cc: `RunOCCScheduler` method (Part 2), `RunOCCParallelScheduler` method (Part 3), `RunMVCCScheduler` method (Part 4), and `RunMVCCSSIScheduler` (Part 5).
* txn/mvcc_storage.cc: `Read` method (Part 4&5), `Write` method (Part 4&5), and `CheckKey` method (Part 4&5).

However, to understand what's going on in the framework, you will need to look through most of the files in the `txn/` directory. We suggest looking first at the **TxnProcessor** class (`txn/txn_processor.h`) and in particular the **RunSerialScheduler()** and **RunLockingScheduler()** methods and examining how it interacts with various objects in the system.


### Part 1A: Simple Locking (exclusive locks only) (10 points)

> 10 points for Part 1A.

Once you've looked through the code and are somewhat familiar with the overall structure and flow, you'll implement a simplified version of two-phase locking. The protocol goes like this:

1. Upon entering the system, each transaction requests an EXCLUSIVE lock on EVERY item that it will either read or write.
2. Wait until all requests have been granted. 
3. Execute the program logic.
4. Release ALL locks at commit/abort time.

In order to avoid the complexities of creating a thread-safe lock manager in this assignment, our implementation only has a single thread that manages the state of the lock manager. This thread performs all the lock requests on behalf of the transactions and then hands over control to a separate execution thread in step (3) above. Note that for workloads where transactions make heavy use of the lock manager, this single lock manager thread may become a performance bottleneck as it has to request and release locks on behalf of ALL transactions.

To help you get comfortable using the transaction processing framework, most of this algorithm is already implemented in `RunLockingScheduler()`. Locks are requested and released at all the right times, and all necessary data structures for an efficient lock manager are already in place. All you need to do is implement the **WriteLock**, **Release**, and **Status** methods in the class `LockManagerA`. Make sure you look at the file `lock_manager.h` which explains the data structures that you will be using to queue up requests for locks in the lock manager.

The test file `txn/lock_manager_test.cc` provides some rudimentary correctness tests for your lock manager implementations, but additional tests may be added when we grade the assignment. We therefore suggest that you augment the tests with any additional cases you can think of that the existing tests do not cover.


### Part 1B: Slightly Less Simple Locking (adding in shared locks) (15 points)

> 15 points for Part 1B.

To increase concurrency, we can allow transactions with overlapping readsets but **disjoint** write sets to execute concurrently. We do this by adding in **SHARED** locks. Again, all data structures already exist, and all you need to implement are the **WriteLock**, **ReadLock**, **Release**, and **Status** methods in the class `LockManagerB`.

Again, `txn/lock_manager_test.cc` provides some basic correctness tests, but you should go beyond these in checking the correctness of your implementation.

### Part 2: Serial Optimistic Concurrency Control (OCC) (10 points)

> 10 points for Part 2.

For OCC, you will have to implement the `RunOCCScheduler` method. This is a simplified version of OCC compared to the one presented in the paper. Pseudocode for the OCC algorithm to implement (in the `RunOCCScheduler` method):

```txt
  while (!stopped_) {
    Get the next new transaction request (if one is pending) and pass it to an execution thread.
    Deal with all transactions that have finished running (see below).
  }

  In the execution thread (we are providing you this code):
    Record occ start index
    Perform "read phase" of transaction:
       Read all relevant data from storage
       Execute the transaction logic (i.e. call Run() on the transaction)

  Dealing with a finished transaction (you must write this code):
    // Validation phase:
    Use the data structure in `txn_processor` class to check overlap with
    each record whose key appears in the txn's read and write sets

    // Commit/restart
    if (validation failed) {
      Cleanup txn
      Completely restart the transaction.
    } else {
      Apply all writes
      Mark transaction as committed
      Update relevant data structure
    }

  cleanup txn:
    txn->reads_.clear();
    txn->writes_.clear();
    txn->status_ = INCOMPLETE;

  Restart txn:
    mutex_.Lock();
    txn->unique_id_ = next_unique_id_;
    next_unique_id_++;
    txn_requests_.Push(txn);
    mutex_.Unlock();
```


### Part 3: Optimistic Concurrency Control with Parallel Validation (12 points)

> 12 points for Part 3.

OCC with parallel validation means that the validation/write steps for OCC are done in parallel across transactions. There are several different ways to do the parallel validation -- here we give a simplified version of the pseudocode from the paper, or you can write your own pseudocode based on the paper's presentation of parallel validation and argue why it's better than the ones presented here (see analysis question 4).

> The `util/atomic.h` file contains data structures that may be useful for this section.

Pseudocode to implement in `RunOCCParallelScheduler`:

```txt
  while (!stopped_) {
    Get the next new transaction request (if one is pending) and pass it to an execution
    thread that executes the txn logic *and also* does the validation and write phases.
  }

  In the execution thread:
    Record start time
    Perform "read phase" of transaction:
       Read all relevant data from storage
       Execute the transaction logic (i.e. call Run() on the transaction)
    <Start of critical section>
    Make a copy of the active set save it
    Add this transaction to the active set
    <End of critical section>
    Do validation phase:
      Use the data structure in `txn_processor` class to check overlap with
      each record whose key appears in the txn's read and write sets

      for (each txn t in the txn's copy of the active set) {
        if (txn's write set intersects with t's write sets) {
          Validation fails!
        }
        if (txn's read set intersects with t's write sets) {
          Validation fails!
        }
      }

      if valid :
        Apply writes;
        Remove this transaction from the active set
        Mark transaction as committed
        Update relevant data structure
      else if validation failed:
        Remove this transaction from the active set
        Cleanup txn
        Completely restart the transaction

    cleanup txn:
       txn->reads_.clear();
       txn->writes_.clear();
       txn->status_ = INCOMPLETE;

    Restart txn:
      mutex_.Lock();
      txn->unique_id_ = next_unique_id_;
      next_unique_id_++;
      txn_requests_.Push(txn);
      mutex_.Unlock();
```

### Part 4: Multiversion Timestamp Ordering Concurrency Control (10 points)

> 10 points for Part 4.

For MVCC, you will have to implement the `TxnProcessor::RunMVCCScheduler` method based on the pseudocode below. The pseudocode implements a simplified version of MVCC relative to the material we presented in class.

Although we give you a version of pseudocode, if you want, you can write your own pseudocode and argue why it's better than the code presented here (see analysis question 6).

In addition you will have to implement the **MVCCStorage::Read**, **MVCCStorage::Write**, **MVCCStorage::CheckWrite**.

Pseudocode for the algorithm to implement (in the `RunMVCCScheduler` method):

```txt
  while (!stopped_) {
    Get the next new transaction request (if one is pending) and pass it to an execution thread.
  }

  In the execution thread:
    Read all necessary data for this transaction from storage (Note that unlike the version of MVCC from class, you should lock the key before each read)
    Execute the transaction logic (i.e. call Run() on the transaction)
    Acquire all locks for keys in the write_set_
    Call MVCCStorage::CheckWrite method to check all keys in the write_set_
    If (each key passed the check)
      Apply the writes
      Release all locks for keys in the write_set_
    else if (at least one key failed the check)
      Release all locks for keys in the write_set_
      Cleanup txn
      Completely restart the transaction.

  cleanup txn:
    txn->reads_.clear();
    txn->writes_.clear();
    txn->status_ = INCOMPLETE;

  Restart txn:
    mutex_.Lock();
    txn->unique_id_ = next_unique_id_;
    next_unique_id_++;
    txn_requests_.Push(txn);
    mutex_.Unlock();
```


### Part 5: Multiversion Concurrency Control with Serializable Snapshot Isolation (2 points)

> 2 points for Part 5.

The MVCC transaction processor you implemented in Part 4 does not produce a serial order. As some of you may have noticed,
MVCC-TO does not validate reads. Therefore a transaction may read a value that was committed by another transaction after it started. PostgreSQL addresses this issue using Serializable Snapshot Isolation (SSI) technique. In this part, you will implement a simplified version of SSI. 

Note: Once you have implemented MVCC without SSI, this part only requires a two line change. 

Pseudocode for the algorithm to implement (in the `RunMVCCSSIScheduler` method):

```txt
  while (!stopped_) {
    Get the next new transaction request (if one is pending) and pass it to an execution thread.
  }

  In the execution thread:
    Read all necessary data for this transaction from storage (Note that unlike the version of MVCC from class, you should lock the key before each read)
    Execute the transaction logic (i.e. call Run() on the transaction)
    Acquire all locks for keys in the read_set_ and write_set_ (Lock any overlapping key only once.)
    Call MVCCStorage::CheckKey method to check all keys
    If (each key passed the check)
      Apply the writes
      Release all locks for all keys
    else if (at least one key failed the check)
      Release all locks for all keys
      Cleanup txn
      Completely restart the transaction.

  cleanup txn:
    txn->reads_.clear();
    txn->writes_.clear();
    txn->status_ = INCOMPLETE;

  Restart txn:
    mutex_.Lock();
    txn->unique_id_ = next_unique_id_;
    next_unique_id_++;
    txn_requests_.Push(txn);
    mutex_.Unlock();
```

### Part 6: Analysis (41 points)

> 3-8 points for each response, total 33 points.

After implementing both locking schemes, both OCC schemes, and the MVCC scheme, please respond to the following questions in a separate file **{your name}-a2-sol.pdf**.

#### 1. Carpe datum (0 points)


When you finish the coding part of this assignment, we will run your code on our test server, and commit the results back to you.
Please note, we will only execute Part 1 - 4 for these experiments. We will not grade you on the performance of Part 5.

#### 2. Simulations are doomed to succeed. (4 points)

Transaction durations are accomplished simply by forcing the thread executing each transaction to run in a busy loop for approximately the amount of time specified. This is supposed to simulate transaction logic --- e.g. the application might run some proprietary customer scoring function to estimate the total value of the customer after reading in the customer record from the database. Please list **at least two weaknesses** of this simulation --- i.e. give two reasons why performance of the different concurrency control schemes you experimented with for this assignment would change relative to each other if we ran actual application code instead of just simulating it with a busy loop.

#### 3. Locking manager (6 points)

- Is deadlock possible in your locking implementation? Why or why not? 
- Most 2PL systems wait until they access data before they request a lock. But this implementation requests all locks before executing any transaction code. What is a performance-related disadvantage of our implementation in this assignment of requesting all locks in advance? What is a client-usability disadvantage of requesting all locks in advance?

#### 4. OCCam's Razor (6 points)

The OCC with serial validation is simpler than OCC with parallel validation.  

- How did the two algorithms compare with each other in this simulation? Why do you think that is the case?
- How does this compare to the OCC paper that we read for class?
- What is the biggest reason for the difference between your results and the what you expected after reading the OCC paper?

#### 5. OCC vs. Locking B  (7 points)

- If your code is correct, you probably found that relative performance of OCC and Locking B were different from the tradeoffs we discussed in class. In fact, you might be quite surprised by your results. Please describe the two biggest differences between the relative performance of OCC vs. Locking B relative to what we discussed in class, and explain why the theory doesn't match the practice for this codebase for each of these two surprises. 

#### 6. MVCC vs. OCC/Locking (6 points)

- For the read-write tests, MVCC performs worse than OCC and Locking. Why? 
- MVCC even sometimes does worse than serial. Why?
- Yet for the mixed read-only/read-write experiment it performs the best, even though it wasn't the best for either read-only nor read-write. Why?

If you wrote your own version, please explain why it's better than the ones presented here.

#### 7. MVCC pseudocode (4 points)

- Why did our MVCC pseudocode request read locks before each read?
- In particular, what would happen if you didn't acquire these read locks?
- How long do these locks have to be held?
- How does the MVCC-SSI pseudocode guarantee serializability?

#### 8. Mixed transaction lengths (8 points)

Take a close look at your high contention read/write results. When transaction lengths are uniform, one of the concurrency control schemes you implemented is best. However, for mixed transaction lengths (the fourth column in the results), you will probably see a different concurrency control scheme as the winner! Please explain why the results changed for mixed transaction lengths.  

### Hints

1. In order to implement OCC, you should take a  look at `committed_txns_`.

2. Inside your `Execution` method of MVCC:  when you call `read()` method to read values from database, please don't forget to provide the third parameter(txn->unique_id_), otherwise the default value is 0 and you always read the oldest version.

3. For MVCC, the performance would be much better if you organize the versions in **decreasing order**. You can implement this by inserting the new version into the right place inside the `write()` method.

## Submission

Please submit **{your_name}_a2_sol.pdf** to the assignment 2 link on ELMS. You do not need to submit any code, since we have access to your repositories.