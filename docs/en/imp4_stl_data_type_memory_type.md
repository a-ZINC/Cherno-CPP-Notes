# STL Containers & Data Types: Comparison Guide

This document organizes common C++ Standard Template Library (STL) containers and related types, grouping them by their functional "siblings" and identifying their primary data storage locations.

### Container & Type Comparison Table

| Category | Type A | Type B | Storage Location |
| --- | --- | --- | --- |
| **Fixed Arrays** | `std::array<T, N>` | C-style `T[]` | Stack (usually) |
| **Dynamic Sequences** | `std::vector<T>` | `std::deque<T>` | Heap |
| **Linked Structures** | `std::list<T>` | `std::forward_list<T>` | Heap |
| **Associative (Ordered)** | `std::map<K, V>` | `std::set<K>` | Heap |
| **Associative (Unordered)** | `std::unordered_map` | `std::unordered_set` | Heap |
| **String Types** | `std::string` | `const char*` / `char[]` | `string` = Heap; `char[]` = Stack |
