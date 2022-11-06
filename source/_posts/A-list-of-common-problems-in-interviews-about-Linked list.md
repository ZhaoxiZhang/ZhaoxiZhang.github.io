---
title: 链表专题——面试中常见的链表问题
date: 2018-10-26
tags: [Interviews,Linked list]
toc: true
---


**声明：**链表定义如下：
<!--more-->
```Java
//Java:
class ListNode {
    int val;
    ListNode next = null;

    ListNode(int val) {
        this.val = val;
    }
}
```
```C++
//C++:
typedef struct ListNode {
    int val;
    struct ListNode *next;
    ListNode(int x) :
        val(x), next(NULL) {
    }
}ListNode;
```

&nbsp;

### 从无头单链表中删除节点
**详情：**给定一个没有头指针的单链表，一个指针指向此单链表中间的一个节点（不是第一个，也不是最后一个节点），请将该节点从单链表中删除。
**题解：**
解法一：由于单链表并没有给出头指针，因此我们无法通过遍历链表的方式找到该节点的前一个节点来改变其 next 指向去指向该节点的 next 节点。换一种思路，我们可以将该节点的元素值全部替换成其 next 节点，然后删除 next 节点，这样就相当于把该节点删除了。
```Java
//Java
public void deleteRandomNode(ListNode currentNode) {
    ListNode nextNode = currentNode.next;
    if (nextNode != null) {
        currentNode.val = nextNode.val;
        currentNode.next = nextNode.next;
    }
    nextNode = null;
}
```
```C++
//C++
void deleteRandomNode(ListNode *current){
    ListNode *next = current->next;
    if (next != NULL){
        current->val = next->val;
        current->next = next->next;
    }
    delete next;
}
```

&nbsp;

### 反转链表
**详情：**给定一个链表的头指针，要求只遍历一次，将单链表中的元素顺序反转过来。
**题解：**
解法一：题目较为简单，每次反转的时候记录下一个节点的指针
```Java
//Java
public ListNode ReverseList(ListNode head) {
    ListNode pre = null, next = null;
    while (head != null) {
        next = head.next;
        head.next = pre;
        pre = head;
        head = next;
    }
    return pre;
}
```
```C++
//C++
ListNode *ReverseList(ListNode *pHead) {
    ListNode *current = NULL, *prev = NULL;
    while (pHead != NULL) {
        current = pHead;
        pHead = pHead->next;
        current->next = prev;
        prev = current;
    }
    return current;
}
```

&nbsp;

### 两个链表的第一个公共节点
**详情：**输入两个链表，找出它们的第一个公共节点
**题解：**
解法一：为了找到两个链表的公共节点，那么我们可以从尾往头遍历查找，但是只给了我们头节点，因此类似于栈的先进后出，因此我们可以用两个栈来保存节点，然后从栈中取出节点进行比较。
解法二：统计两个链表的长度 len1 和 len2，让较长的链表先走`abs(len1 - len2)`长度，之后二者同时继续往下遍历，查找第一个公共节点。

```C++
//C++
ListNode *FindFirstCommonNode( ListNode *pHead1, ListNode *pHead2) {
    int len1 = SizeLinkedList(pHead1);
    int len2 = SizeLinkedList(pHead2);

    if (len1 > len2) {
        pHead1 = walker(pHead1, len1 - len2);
    } else {
        pHead2 = walker(pHead2, len2 - len1);
    }

    while (pHead1->val != pHead2->val) {
        pHead1 = pHead1->next;
        pHead2 = pHead2->next;
    }

    return pHead1;
}

int SizeLinkedList(ListNode *head) {
    if (head == NULL)   return 0;
    int size = 0;
    ListNode *current = head;
    while (current != NULL) {
        size++;
        current = current->next;
    }
    return size;
}

ListNode *walker(ListNode *head, int cnt) {
    while (cnt--) {
        head = head->next;
    }
    return head;
}
```

&nbsp;

### 判断给定链表是否存在环
**详情：**给定一个链表，判断这个链表是否存在环
**题解：**
解法一：[Floyd判圈算法](https://www.cnblogs.com/ZhaoxiCheung/p/7355369.html)
```C++
//C++
bool hasRing(ListNode *pHead){
    bool hasRing = false;
    ListNode *fast = pHead, *slow = pHead;
    while (fast != NULL && fast->next != NULL){
        fast = fast->next->next;
        slow = slow->next;
        if (fast == slow)   hasRing = true;
    }
    return hasRing;
}
```

&nbsp;

### 链表中环的入口节点
**详情：**给一个链表，若其中包含环，请找出该链表的环的入口结点，否则，输出null。
**题解：**
解法一：[Floyd判圈算法](https://www.cnblogs.com/ZhaoxiCheung/p/7355369.html)
```Java
//Java
public ListNode EntryNodeOfLoop(ListNode pHead) {
    ListNode fast = pHead, slow = pHead;
    while (fast != null && fast.next != null) {
        fast = fast.next.next;
        slow = slow.next;
        if (fast == slow)   break;
    }
    if (fast == null || fast.next == null)  return null;
    fast = pHead;
    while (fast != slow) {
        fast = fast.next;
        slow = slow.next;
    }

    return fast;
}
```
```C++
//C++
ListNode *EntryNodeOfLoop(ListNode *pHead) {
    ListNode *slow = pHead, *fast = pHead;
    while (fast != NULL && fast->next != NULL) {
        fast = fast->next->next;
        slow = slow->next;
        if (fast == slow)   break;
    }
    if (fast == NULL || fast->next == NULL)   return NULL;
    fast = pHead;
    while (fast != slow) {
        fast = fast->next;
        slow = slow->next;
    }
    return fast;
}
```

&nbsp;

### 判断两个链表是否相交
**详情：**给定两个单链表的头指针，判断这两个链表是否相交。
**题解：**
解法一：若两个链表相交，则链表的最后一个节点一定是公共的，因此可以利用这个性质求解。
```C++
//C++
bool isIntersect(ListNode *pHead1, ListNode *pHead2){
    if (pHead1 == NULL || pHead2 == NULL)   return false;
    while (pHead1->next != NULL)    pHead1 = pHead1->next;
    while (pHead2->next != NULL)    pHead2 = pHead2->next;
    if (pHead1 == pHead2)   return true;
    return false;
}
```
解法二：由于都是单项链表，也就是都没有环，那么我们可以把第一个链表链接到第二个链表后面，如果新的链表有环，证明了有公共节点。
```C++
//C++
bool isIntersect(ListNode *pHead1, ListNode *pHead2){
    if (pHead1 == NULL || pHead2 == NULL)   return false;
    pHead1->next = pHead2;
    return hasRing(pHead1);
}
```

&nbsp;

### 判断两个链表是否相交**变形**
**详情：**给定两个**有环**链表的头指针，判断这两个链表是否相交。
**题解：**
解法一：对于有环链表，如果相交，存在以下几种情况：
![](https://img2018.cnblogs.com/blog/885804/201810/885804-20181026172408920-46021901.png)
因此，找到链表的入口节点，判断是否相等，对应情形一和二，对于三，我们可以固定一个节点，然后遍历链表来判断是否存在相交。
```C++
//C++
bool isIntersect(ListNode *pHead1, ListNode *pHead2){
    if (pHead1 == NULL || pHead2 == NULL)   return false;
    ListNode *entry1 = EntryNodeOfLoop(pHead1);
    ListNode *entry2 = EntryNodeOfLoop(pHead2);

    if (entry1 == entry2)   return true;
    else{
        ListNode *backup = entry2;
        do
        {
            entry2 = entry2->next;
        }while (entry2 != entry1 && entry2 != backup);
        return entry2 != backup;
    }
}
```

&nbsp;

### 合并两个排序的链表
**详情：**输入两个单调递增的链表，输出两个链表合成后的链表，当然我们需要合成后的链表满足单调不减规则。
**题解：**
解法一：
```Java
//Java
public ListNode Merge(ListNode list1, ListNode list2) {
    if (list1 == null) {
        return list2;
    }
    if (list2 == null) {
        return list1;
    }

    ListNode prev = null;
    ListNode root = list1.val < list2.val ? list1 : list2;
    while (list1 != null && list2 != null) {
        if (list1.val < list2.val) {
            if (prev == null) {
                prev = list1;
            } else {
                prev.next = list1;
                prev = list1;
            }
            list1 = list1.next;
        } else {
            if (prev == null) {
                prev = list2;
            } else {
                prev.next = list2;
                prev = list2;
            }
            list2 = list2.next;
        }
    }
    while (list1 != null) {
        prev.next = list1;
        prev = list1;
        list1 = list1.next;
    }
    while (list2 != null) {
        prev.next = list2;
        prev = list2;
        list2 = list2.next;
    }

    return root;
}
```
```C++
//C++
ListNode *Merge(ListNode *pHead1, ListNode *pHead2) {
    if (pHead1 == NULL) return pHead2;
    if (pHead2 == NULL) return pHead1;
    ListNode *prev = NULL;
    ListNode *root = pHead1->val < pHead2->val ? pHead1 : pHead2;
    while (pHead1 != NULL && pHead2 != NULL) {
        if (pHead1->val < pHead2->val) {
            if (prev == NULL) {
                prev = pHead1;
            } else {
                prev->next = pHead1;
                prev = pHead1;
            }
            pHead1 = pHead1->next;
        } else {
            if (prev == NULL) {
                prev = pHead2;
            } else {
                prev->next = pHead2;
                prev = pHead2;
            }
            pHead2 = pHead2->next;
        }
    }

    while (pHead1 != NULL) {
        prev->next = pHead1;
        prev = pHead1;
        pHead1 = pHead1->next;
    }
    while (pHead2 != NULL) {
        prev->next = pHead2;
        prev = pHead2;
        pHead2 = pHead2->next;
    }
    return root;
}
```