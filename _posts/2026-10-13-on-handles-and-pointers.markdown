# On handles and pointers

I've recently looked into using handles instead of pointers in my programming[1](#footnote1) based on ideas from the article [Handles are the better pointers](https://floooh.github.io/2018/06/17/handles-vs-pointers.html).

My reason for doing this is that I'm writing a garbage collector for a small script language I'm creating. The current code is based on [polymorphic memory resources](https://en.cppreference.com/cpp/header/memory_resource) and uses `std::allocate_shared` to create a shared_ptr that I save into a vector.
This works well, the shared_ptr is a bit of a hack since I expose `T*` outside the GC but `std::allocate_shared` is too convenient for using pmr for me to care. Pmr also makes memory tracking easy, useful for triggering GC, and monotonic buffer resource and unsynchronized pool resource makes allocations amortized O(1).

This is enough for my language but I can't help to notice small annoyances with this that I would like to fix. An obvious problem is that the allocator can't move objects but also things like `List` and `String` are different enough to potentially want different allocation strategies.

That's where using handles comes in. Now the technique isn't new even to me, but I haven't gone all in with it before.

## What is a handle?
In its most basic form a handle is just an index into an array of a specific type. This already is enough to create a number of benefits.

Checking for overflow is trivial. You just need to check if the index lies within `0 <= index < array_length`.

Type safety. One of the dangerous things with normal use after free is that you allow an attacker-controlled type to behave as another type in its memory location. Handles don't prevent this but it makes the attack harder to execute and can be thought of as a defense in depth technique.

Moving memory. This is where I'm coming from. By moving memory around to an optimal layout you can make allocations faster and remove fragmentation. Also by moving the memory you can make sure that it's laid out in a cache friendly way tailored to your type.

## What you're giving up.
The one that's hardest for me is RAII. Now not all is lost and I have a `unique_handle` that frees on destruction but it doesn't play as nice with handles as it does pointers. There are also ways of creating reference counted and weak handles but I've yet to try it.

Since RAII becomes harder to write, writing code using exceptions becomes even harder. I know not everyone likes them but I do, so into the negative bucket this goes.

Overhead. It's more code to write when you must access an array before being able to use the memory. Less of a problem but worth mentioning is that it's also more overhead for the CPU.

Variable length data. While not impossible, handles work best when the data is a single unified type.

Sanitizers and other tools. As far as valgrind is concerned the memory in your arrays never leaks. Similarly you cannot have a use after free index if your array is alive as long as your progress, at least from the point of view of the compiler and runtime. One of the reasons OpenSSL Heartbleed went on undetected is that the memory went through their own `OpenSSL_malloc()` allocator.

## Other benefits of handles 
I've already mentioned the dreaded use after free vulnerability. By using handles you can easily detect it. One thing you can try is to scramble or overwrite the memory during release. This will protect you from issues similar to the famous OpenSSL heartbleed vulnerability. You could also have a `boolean` `in_use` set to true when acquiring the resource and set it to false when releasing. Since a `boolean` takes up one byte of memory while logically only one bit you can also pass along a generation count to see which allocation was the one that was dangling.

They can take up less space. A pointer is normally 64-bit and allows you to access the entire memory space. However if you only need 256 entities you can save up to 56-bit. This small improvement can have a significant difference if you have a lot of small data that you can fit in the CPU cache.

If your handles are only used during compile time you can make them more readable by giving them a name. A simple consteval hash function can make sure that your handles get a unique index by hashing them. Your code would look something like:
```cpp
constexpr Handle operator""_handle(const char* text, std::size_t length)
{
  return sha512(text, length);
}
auto h = gh.acquire("GitHub access token"_handle); auto h = db.lookup("Database table"_handle);
```

Type specification. If your entities are similar but different (think classic OOP inheritance) you can use the extra unused bits in the handle to differentiate the entities. A simple `switch` statement can be used to direct the handle to the correct type array.

You can split the entities in any way you want. Often our data structures contain more data than we need for all operations. By splitting an entity we can separate data that is hot and frequently accessed, like position and velocity, from data that is accessed infrequently and cold, like usernames.

We can optimize the algorithms we use to take advantage of how we place the data. Using classic Struct of Arrays (SoA) over Array of Structs (AoS) patterns make it easy to use SIMD in our algorithm. Similarly we can optimize and use data structures to prefer reading or writing, depending on our access patterns.

We can decide if we want our arrays dense or sparse. Many entities don't need all the available properties. We can store a bitmask about which components an entity is using and with another indirection array we can save only those. Or if most entities use most properties we can use a little bit of extra memory to store that data even if it's unused. We only need to store that information somewhere (like the afford mentioned bitmask). Of course we can also use both methods if the data access and coding effort warrants it.

Specialized memory allocators. While handles aren't a requirement here, I use PMR with normal STL operations as mentioned above, it's more obvious with handles that you can use something like a bump allocator. It's also easier to write it if you only have to worry about a single type.

Multithreding: if we have *N* cores one core can work on `array_size / N` items. If we have multiple arrays, each core can work on a different array.

## How to write code using handles instead of pointers
We'll make a number of simplifications while writing our pseudocode. However that only proves the benefit of handles since you decide which simplification or extra complexity is worth it.

First I'll use a dynamically allocated `std::vector`. Second I won't preallocate memory for it despite having a good initial idea of the number of entities. I'll limit the maximum number of entities to slightly more than 4.2 billion, fitting a 32-bit unsigned integer. I'll use a 32-bit generation count despite it being more than we need. I'll use global variables instead of proper encapsulation. I'll use sparse arrays rather than dense. I also use a simple free list for reusing released handles. I keep the generation counts in their own array since they're colde data and increase the, in the release function.

I'm also making the common mistake of using game development for my example. Yes, this is useful for games but it is useful for other types of applications as well.

```cpp
// Hot data used in regularly run loops
struct MyEntityPosition {
	float x, y;
	float dx, dy;
};

// Cold data that's looked up occasionally
struct MyEntityColdData {
	int current_hp;
	int max_hp;
	string name;
	map<string, any> custom_properties; // just some cold data that is expensive to reallocate
};

// Two different arrays for two different types
// of memory access.
static vector<MyEntitiesPosition> gs_EntityPosition;
static vector<MyEntityColdData> gs_EntitiesData;

// Handle to our entity. This could be two uint16_t
// if we don't need more than 65535 entities.
struct EntityHandle {
	uint32_t index;
	uint32_t generation = 0; // 0 == invalid generation
};

// Bookkeeping arrays
static vector<uint32_t> gs_Generations;
static vector<uint32_t> gs_FreeEntities;

// Our arrays must be the same size.
assert(gs_EntityPosition.size() == gs_EntitiesData.size());
assert(gs_EntitiesData.size() == gs_Generations.size());

// Check if we can reuse an already free()'d handle
// otherwise create a new and use that one.
EntityHandle acquire() {
  uint32_t index = 0;
  if (gs_FreeEntities.empty()) {
    index = gs_EntitiesData.size();
    gs_EntitiesPosition.emplace_back();
    gs_EntitiesData.emplace_back();
    gs_Generations.emplace_back();
    gs_Generations.back() = 1;
  } else {
    index = gs_FreeList.back();
    gs_Entities.pop_back();
  }

  // Skip initialization for now.
  return {index, gs_Generations[index]);
}

// Update the positions in the hot path.
// This could use SoA and SIMD for extra
// performance.
void movePositions() {
  for (auto& pos : gs_EntityPosition) {
    pos.x += pos.dx;
    pos.y += pos.dy;
  }
}

// Simple property lookup.
// We assert that the handle is valid here.
int currentHp(EntityHandle h) {
  assert(h.index < gs_EntityData.size());
  assert(h.generation == gs_Generations[h.index]);
  return gs_EntityData[h.index];
}

// Release the handle and put it in the free list
// after increaing the generation count.
//
// Releasing an invalid entity is safe so we just
// log it and return.
void releaseEntity(EntityHandle h) {
  if (h.generation == 0) {
    log("Error. Entity of invalid generation released");
    return;
  }
  if (h.index >= gs_EntityData.size()) {
    log("Error handle (%d) out-of-bounds (%d)", h.index, gs_EntityData.size());
    Return;
  }
  if (h.generation != gs_Generations[h.index]) {
    log("Error.Entity of generation %d released. Current generation %d", h.generation, gs_Generations[h.index]);
    return;
  }

  // Scramble data
  gs_EntityPosition[h.index].x = 0xFFFF'FFFF;
  gs_EntityPosition[h.index].y = 0x0000'0000;
  gs_EntitiesData[h.index].current_hp = 0xAAAA'AAAA;
  gs_EntitiesData[h.index].max_hp = 0x9999'9999;
  gs_EntitiesData[h.index] = "<redacted>";

  gs_Generations[h.index]++;
  if (gs_Generations[h.index] == 0) {
    // 0 == invalid generation
    gs_Generations[h.index]++;
  }

  gs_FreeList.push_back(h.index);
}
```

<a id="footnote1"></a> [1]: If you've used DDD value objects for backend programming, and its us of handles you've done a variety of this. You just used it for handling database layout instead of raw memory layout.

