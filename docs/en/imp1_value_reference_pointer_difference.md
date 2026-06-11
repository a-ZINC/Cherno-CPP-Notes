## C++: Value vs. Reference vs. Pointer

| Feature | Value (`T obj`) | Reference (`T& ref`) | Pointer (`T* ptr`) |
| --- | --- | --- | --- |
| **Constructor Call** | Calls Copy/Move Constructor | No constructor called | No constructor called |
| **Memory Location** | Usually Stack | Aliases existing memory | Stores an address (Stack/Heap) |
| **Polymorphism** | No (Object Slicing) | Yes (if `virtual`) | Yes (if `virtual`) |
| **Nullability** | Cannot be null | Cannot be null | Can be `nullptr` |

---

### 1. Construction and Slicing

When you assign by **value**, you trigger a copy or move constructor because you are creating a *new* object. References and pointers are simply different ways to access the existing object, so they do not trigger constructors.

```cpp
Entity e1("Base");
Entity e2 = e1; // Calls Copy Constructor (Value)

Entity& ref = e1; // No constructor (Reference)
Entity* ptr = &e1; // No constructor (Pointer)

```

---

### 2. Stack vs. Heap Instantiation

* **Stack:** Automatically managed, faster, destroyed when the scope ends.
* **Heap:** Manually managed via `new`, persists until `delete` is called.

```cpp
// Stack instantiation
Entity e; 

// Heap instantiation
Entity* h = new Entity("Heap");
delete h; // Must delete to avoid memory leak

```

---

### 3. Polymorphism and Object Slicing

When you assign a `Player` (child) to an `Entity` (parent) by **value**, the child-specific parts are "sliced off." Pointers and references keep the original object's identity, allowing `virtual` functions to work.

#### Example Code:

```cpp
#include <iostream>
#include <string>

class Entity {
public:
    virtual std::string getName() { return "Entity"; }
};

class Player : public Entity {
public:
    std::string getName() override { return "Player"; }
};

int main() {
    Player p;
    
    // 1. Value (Slicing occurs)
    Entity e1 = p; 
    std::cout << e1.getName() << std::endl; // Output: Entity

    // 2. Pointer (Polymorphism preserved)
    Entity* e2 = &p;
    std::cout << e2->getName() << std::endl; // Output: Player

    // 3. Reference (Polymorphism preserved)
    Entity& e3 = p;
    std::cout << e3.getName() << std::endl; // Output: Player

    return 0;
}

```

#### Output:

```text
Entity
Player
Player

```

---

### Summary Note

* **Value:** Use when you want a completely independent copy of an object.
* **Reference:** Use when you want to pass an object to a function or alias it without copying and without needing null-safety.
* **Pointer:** Use when you need to represent optionality (nullability) or when you need to change which object the variable points to at runtime.
