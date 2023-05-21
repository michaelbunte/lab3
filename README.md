# Hash Hash Hash
In this lab we experiment with placing locks in different parts of a hash
table to enable concurrent access. Without carefully placing locks, different
processes can access the hash table in damaging ways to destroy the files. 

## Building
```shell
make
```

## Running
```shell
./hash-table-tester
```

## First Implementation
We initialize a lock at the beginning of our c file with:  
`static pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;`

In the `hash_table_v1_add_entry` function, we simply access this lock at
the beginning of the function with
`int e = pthread_mutex_lock(&mutex); if(e != 0) exit(errno);`

Once we have added (or updated) an entry, we can free it to let another 
process access it with 
`e = pthread_mutex_unlock(&mutex); if( e != 0 ) exit(errno);`

The reason this works is because the only times we actually update our 
table is when we add an object to the table. If we lock the 
`hash_table_v1_add_entry` function, then no other functions are able to access
and manipulate the hash table, and we won't run into conflicts

### Performance
```shell
$ ./hash-table-tester -t 4 -s 25000
Hash table v1: 126,991 usec
  - 0 missing
```

This time version 1 is 124,418 usec

## Second Implementation

This time, instead of putting a lock around the entire hash table, I added
a lock inside of each hash_table_entry:

```c
struct hash_table_entry {
	pthread_mutex_t entry_lock;
	struct list_head list_head;
};
```
This allows us to lock of hash_table_entries separately instead of the entire
hash table.

Therefore, we can actually add elements at the same time to the hash table,
given that their key's hash to different values. 

We do this by adding a lock 

In the `hash_table_v2_add_entry` function, in the case where we update and
access an list_entry, can add locks to now say
```c
if (list_entry != NULL) {
		int e = pthread_mutex_lock(&hash_table_entry->entry_lock);
		if(e != 0) exit(errno);

		list_entry->value = value;

		e = pthread_mutex_lock(&hash_table_entry->entry_lock);
		if(e != 0) exit(errno);

		return;
	}
```
Now if the list_entry isn't NULL, (AKA the entry doesn't exist), then we 
add locks again to say

```c
int e = pthread_mutex_lock(&hash_table_entry->entry_lock);
if(e != 0) exit(errno);

list_entry = calloc(1, sizeof(struct list_entry));
list_entry->key = key;
list_entry->value = value;
SLIST_INSERT_HEAD(list_head, list_entry, pointers);

e = pthread_mutex_unlock(&hash_table_entry->entry_lock);
if(e != 0) exit(errno);
```
Both of these situations work because we have isolated the bucket of the hash 
table in order to either update an existing value, or add another item to 
the bucket

### Performance
```shell
$ ./hash-table-tester -t 4 -s 25000
Hash table v2: 32,147 usec
  - 0 missing
```

This time the speed up is 126991/32147 = 3.95 times faster

This is faster because in `v1`, when we wanted to add an element to the hash
table, we had to lock the entire hash table. This would mean that when multiple
processes attempted to add elements to the hash table, they ended up having to
wait. This was very slow.

On the other hand, with `v2`, this was much faster because we only would lock
the bucket of the hash table that was being edited. This was much faster because
generally when different threads would access an element of the hash table, they
would be interacting with entirely different buckets, so there wouldn't be 
collisions. 

With this scenario here, there still are collisions (if two threads attempt) to
access two elements that happen to hash to the same bucket of the table, but
the chance of this is much lower, and thus the hash table is much more efficient

## Cleaning up
```shell
make clean
```