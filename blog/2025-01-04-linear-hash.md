# Understanding Linear Hash Step by Step

Many computer science concepts seem complex at first; however, if we break them down step by step, they become much clearer. In this article, I will describe how a hash table works, how resizing works, and how linear hashing improves resizing. By the end, you will see that linear hashing is straightforward to understand.

## Hash Table
A hash table uses an array to store key-value pairs. When we insert a pair, we use a hash function to calculate the key’s index and then insert the pair at that index in the array.

```ruby
def insert(key, val)
   index = hash(key) % @array.length
   @array[index] = [key, val]
end
```

Since two different keys may have the same hash value (`hash(key1) == hash(key2)`), we handle this by using a linked list to store multiple pairs at the same index.

```ruby
def insert(key, val)
  index = hash(key) % @array.length
  node = Node.new(key, val)
  if @array[index]
    @array[index].next = node
  else
    @array[index] = node
  end
end
```

## Resizing & Linear Hash
From the code above, we see that if a newly inserted key’s hash value matches an existing key’s hash value, the new pair is appended to the linked list at that index. If such collisions occur frequently, the linked list grows longer, and the hash table becomes slower.

The most simple solution is to double the array size. But after doubling the array size, we must rearrange the key-value pairs.

Suppose the array’s length is 4, with key1’s hash value being 0 and key2’s hash value being 4. Since the array’s length is 4, both of these keys are stored at index 0. After doubling the array size, key1 should be stored at index 0 and key2 at index 4. After resizing, we must rearrange these pairs. This process is called resizing or rehashing.

<img width="360" alt="Screenshot 2025-01-03 at 00 27 45" src="https://github.com/user-attachments/assets/a54bbf13-a4b2-41dc-92ae-1bf26eb9b911" />

This basic resizing algorithm resolves the conflict issue, but if the array becomes very large, we must move many pairs to their new locations, which can consume much CPU in a short time.

To improve this, instead of doubling the array size, we could increase the size one step at a time. For example, we add one element to the array at a time and move the corresponding pairs to the new array item. Now, we need to consider how to calculate the index.

Suppose we use the old hash function `hash(key) % 4` and get an index of 1, 2, or 3. In this case, we know there is no new array item for those keys—they are correctly indexed by the old hash function. However, if the index is 0, after adding the new item, its correct index could be 0 or 4, so we need to use a new hash function `hash(key) % 8` to find the correct index.

With two hash functions (`hash(key) % 4` and `hash(key) % 8`), we need to determine which one to use. The solution is simple: we can maintain a cursor to track which elements have already been resized. First, we use the old hash function to calculate the index. If the index is less than the cursor, it means we have already created a new array element for it, so we use the new hash function. If the index is greater than or equal to the cursor, we know it doesn't have a new array element, so the old hash function is correct.

<img width="748" alt="Screenshot 2025-01-03 at 00 30 34" src="https://github.com/user-attachments/assets/6b73e6fc-4fb9-4cb9-b657-feb1710f8e84" />

```ruby
def index(key)
  i = hash(key) % @n

  if i < @cursor
    i = hash(key) % (@n * 2)
  end

  i
end
```

## Summary
This article doesn't cover all the details, such as how to maintain the `@n`, but I hope it gives you a clear understanding of linear hashing. Learning an algorithm’s implementation may seem complex at first, but once we understand the ideas behind it, everything becomes much clearer.
