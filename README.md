# Notes

## W1 D1

Why do we need a combination of `state` and `state_lock`? Can we only use `state.read()` and `state.write()`?
- First reason. We get the state and check if update is needed. However, during we go from get the `Update_needed` state to check, other threads may get the `Update_needed` state as well. So all these threads will finally update the state after check. In this case, we might create one empty memtable which is then immediately frozen. This can be solved by `mutex`. (see the code)
- Second reason. To separate the expensive write lock and the lock needed for checking. Without `mutex`, we have to do check in the write lock. With `mutex`, we can first `mutex.lock()` the thread that want to do further check, where the write_lock has not been validated. After the check, the write_lock is validated.

## W1 D2
The LSM now has one mutable memtable and many archived memtables. So, each memtable has one iterator and LSM has an iterator by merging all memtable iterators.

The skipmap is sorted, so the memtable iterators are originally sorted. To iterate over all k-v pairs in all memtables, order is defined as 'iterate from bigger key to smaller key, and if a key exists multiple times, get the latest value'. According to this order, memtable iterators are wrapped by a binary heap, and together sent and managed by a merged iterator.

Memtable iter is in `mem_table.rs` called `MemTableIterator`. Merged iter is in `iterators/merge_iterator.rs` called `MergeIterator`. The joint usage of data structure like max-heap and skip-map grant the local order and global order, which is beautiful. You should review it.

Also, every iterator is inherited from an ABC:
```rust
pub trait StorageIterator {
    type KeyType<'a>: PartialEq + Eq + PartialOrd + Ord

    /// Get the current value.
    fn value(&self) -> &[u8];

    /// Get the current key.
    fn key(&self) -> Self::KeyType<'_>;

    /// Check if the current iterator is valid.
    fn is_valid(&self) -> bool;

    /// Move to the next position.
    fn next(&mut self) -> anyhow::Result<()>;

    /// Number of underlying active iterators for this iterator.
    fn num_active_iterators(&self) -> usize {
        1
    }
}
```