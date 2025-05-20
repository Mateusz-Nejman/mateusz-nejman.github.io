---
layout: post
title: Unmanaged memory in C#
description: After few days fight, I think I understood a little bit unmanaged memory in C#
date: 2025-05-16
categories: [Unmanaged, Pointers]
---

After few days fight, I think I understood a little bit unmanaged memory in C#, and I will try to share my knowledge with you.

### Why I used pointers in C#?

For my Pixed App, I tried to manage memory more efficient than garbage collector did. Everytime, when user did changes in image, history service generated copy of project stored in byte arrays. On larger projects, after few minutes of working, memory was filled with byte arrays. Debugger showed me 2-3GB of memory used by my app, which was unacceptable for me. I reimplemented [HistoryService](https://github.com/Mateusz-Nejman/Pixed/blob/master/Pixed.Application/Services/HistoryService.cs) to save history changes to files and clear this files on app exit. It worked fast on Windows, but not on my tablet. The reason was slow IO operation methods on Android. So, I implemented simple cache system, where after increasing history data to 512MB, data will be saved to file from another thread. History service started working faster on Android, but I found, that garbage collector stored my teoretically removed data in memory and it took some time to be finally removed from memory. I decided to use pointers similarly to C++ language. I wanted to clear my memory short after disposing data so I started searching.

### Pros and cons

Pros:

*   Instant memory cleanup
    

Cons:

*   Memory leak if you don't take care of it
    

### Creating unmanaged array

For using unmanaged memory, we need unsafe context. unsafe means that we can use memory directly. For this case you can set "Unsafe code" in project settings.

For creating unmanaged array we need to know size of our array. I didn't found yet how in easy way I can create dynamically unmanaged collection. I will show you how to allocate int array.

```csharp
IntPtr intPtr = Marshal.AllocHGlobal(sizeof(int) * length);

```

Basically, here we have C# equivalent of C++ pointers. Using `Marshal.AllocHGlobal` we are creating pointer to allocated memory where we will store our data. As parameter, we need to pass our length of array multiplied by size of our type. It's because we are allocating byte memory.

For better readability and usability, we can cast pointer to:

```csharp
int* ptr = (int*)intPtr;

```

Now, I want to store some numbers in this array, so for setting data I'm using:

```csharp
*(ptr + offset) = value;

```

Example:

```csharp
for(int offset = 0; offset < length; offset++)
{
    *(ptr + offset) = offset;
}

```

Okay, I added some values to array, but what with reading? Reading is possible using the same way like writing:

```csharp
int value = *(ptr + offset);

```

It's not that hard like I thought when I learned C++ :)

When we don't need this data anymore, we need to clean memory, because garbage collector will not do this for us.

```csharp
Marshal.FreeHGlobal((IntPtr)ptr);

```

### Simple collection class example

```csharp
unsafe class UnmanagedArray<T> : IEnumerable<T>, IDisposable where T : unmanaged
{
    private class UnmanagedEnumerator<T1> : IEnumerator<T1>
    {
        private readonly T1* _ptr;
        private readonly int _length;
        private T1 _current;
        private int _pos;

        public UnmanagedEnumerator(T1* ptr, int length)
        {
            _ptr = ptr;
            _length = length;
            _pos = 0;
            _current = default(T1);
        }

        public T1 Current => _current;

        object IEnumerator.Current => Current;

        public bool MoveNext()
        {
            _pos++;

            if(_pos >= _length)
            {
                _current = default(T1);
                return false;
            }

            _current = _ptr[_pos];
            return true;
        }

        public void Reset()
        {
            _pos = 0;
        }

        public void Dispose()
        {
        }
    }

    private readonly T* _ptr;
    private readonly int _length;
    private bool _disposed;

    public T this[int i]
    {
        get { return *(_ptr + i); }
        set { *(_ptr + i) = value; }
    }

    public int Length => _length;
    public UnmanagedArray(int length)
    {
        IntPtr ptr = Marshal.AllocHGlobal(sizeof(int) * length);
        int* ptr1 = (int*)ptr;

        for(int offset = 0; offset < length; offset++)
        {
            *(ptr1 + offset) = offset;
        }
        _ptr = (T*)Marshal.AllocHGlobal(sizeof(T) * length);
        _length = length;
    }
    ~UnmanagedArray()
    {
        Dispose(disposing: false);
    }

    public IEnumerator<T> GetEnumerator()
    {
        return new UnmanagedEnumerator<T>(_ptr, _length);
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
            }

            Marshal.FreeHGlobal((IntPtr)_ptr);

            _disposed = true;
        }
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
}

```
