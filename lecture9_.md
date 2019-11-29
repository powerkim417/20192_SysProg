# Lecture 9. Memory Management

## Linux Memory Management

### Virtual Address Space

### Physical Memory Layout for 'Kernel Core'

### Physical Memory Mapping in Linux/x86

### Constructing Page Tables

#### Constructing Kernel Page Tables

#### Constructing User Page Tables

## Physical (Dynamic) Memory Management

### Physical Memory Structure

Node, Zone, Page Frame으로 이루어짐

![image-20191125133214013](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191125133214013.png)

#### Page Frames

#### Node

#### Zone

### Process Address Mapping: 요약

## Kernel Memory Allocation(KMA) in Linux

### Contiguous Page Frame Allocator

#### Buddy System

### Noncontiguous Page Frame Allocator

### Memory Object Allocator

#### Slab Allocator

##### 구조

##### Specific Caches

##### General Caches

##### /proc/slabinfo

##### Allocating and Releasing

##### Summary

### Which Method?

