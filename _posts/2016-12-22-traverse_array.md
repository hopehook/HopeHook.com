---
date: 2016-12-22
layout: post
title: 数组遍历:递归深度优先,递归广度优先,非递归深度优先,非递归广度优先
thread: 2016-12-22-traverse_array.md
categories: 算法
tags: python
---


## 1 递归深度优先
* depth.py
<pre>
arr = [1, 2, 3, [[[4]], 5], 6, [7, 8, [9, 10]], [11, [12,[13,[14, 15]]], 16, 17]]


def depth_visit(arr):
        if isinstance(arr, list):
		for child in arr:
		        print child
			depth_visit(child)


depth_visit(arr)
</pre>


## 2 递归广度优先
* breadth.py

<pre>
arr = [1, 2, 3, [[[4]], 5], 6, [7, 8, [9, 10]], [11, [12,[13,[14, 15]]], 16, 17]]

def breadth_vist(arr):
	result = []
	for child in arr:
		print child
		if isinstance(child, list):
			for tmp_child in child:
				result.append(tmp_child)
		
	if len(result) > 0:	
		breadth_vist(result)


breadth_vist(arr)
</pre>

## 3 非递归深度优先
* depth2.py

<pre>
class Stack:
  def __init__(self):
    self.items = []
     
  def isEmpty(self):
    return len(self.items)==0
   
  def push(self, item):
    self.items.append(item)
   
  def pop(self):
    return self.items.pop() 
   
  def peek(self):
    if not self.isEmpty():
      return self.items[len(self.items)-1]
     
  def size(self):
    return len(self.items) 


stack = Stack()



arr = [1, 2, 3, [[[4]], 5], 6, [7, 8, [9, 10]], [11, [12,[13,[14, 15]]], 16, 17]]

for child in arr:
	stack.push(child)
	while(not stack.isEmpty()):
		x = stack.pop()
		print x
		if isinstance(x, list):
			for tmp_child in x:
				tmp_child
				stack.push(tmp_child)
</pre>

## 4 非递归广度优先
* breadth2.py

<pre>
class Queue():  
    def __init__(self):  
        self.items = []  
    
    def empty(self):  
        return self.items == []  
    
    def enqueue(self,data):  
        self.items.append(data)  
    
    def dequeue(self):  
        if self.empty():  
            return None  
        else:  
            return self.items.pop(0)  
    
    def head(self):  
        if self.empty():  
            return None  
        else:  
            return self.items[0]  
    
    def length(self):  
        return len(self.items)



q = Queue()


arr = [1, 2, 3, [[[4]], 5], 6, [7, 8, [9, 10]], [11, [12,[13,[14, 15]]], 16, 17]]


for child in arr:
	print child
	q.enqueue(child)

while(not q.empty()):
	x = q.dequeue()
	if isinstance(x, list):
		for tmp_child in x:
			print tmp_child			
			q.enqueue(tmp_child)

</pre>



