Iterators for Beginners. The Way of С++
#######################################

:date: 2013-01-20
:author: Roman Podolyaka
:tags: cpp, iterators
:category: programming
:slug: iterators-for-beginners-c++
:lang: en


Foreword
--------

This post is mainly for beginners and is not intended to teach you everything
about iterators in ``C++``. So if you know about ``iterator_traits<>`` or something
like that you might get bored reading it. Anyway I hope this information will be
useful for the people who haven't heard about iterators before.


The problem
-----------

Lets consider the following problem: we need a function that would return an index of
the maximum element in a given array of integers. We could implement it like this:

.. code-block:: cpp

   int max_element(int arr[], int len)
   {
       int max = 0;

       for (int i = 1; i < len; ++i)
       {
           if (arr[i] > arr[max])
               max = i;
       }

       return max;
   }

This code works perfectly but it has a huge limitation though - it works only for arrays of ``int`` elements.
To find a maximum element in an array of floats we would have to write completely the same code and
substitute ``int`` with ``float`` in the declaration of an array type. We all know that ``C++`` has
a solution for this problem - *templates*.

A generalized version of the function which returns an index of maximum element in an array of any type
(precisely, for which the relation *greater than* is defined):

.. code-block:: cpp

    template<class T>
    int max_element(T arr[], int len)
    {
        int max = 0;

        for (int i = 1; i < len; ++i)
        {
            if (arr[i] > arr[max])
                max = i;
        }

        return max;
    }

Now we must be satisfied. But what if we wanted to search for the maximum element only in a small part of
the whole array?

Arrays in ``C++`` are linearly allocated memory segments filled with elements of some type with no gaps
between them. Moreover an array name is also a pointer to the first element of this array. Arrays and pointers
have many things in common and, as you may already know, it is perfectly ok to use pointer arithmetic to access
an array elements.

The elements range for the function to process is specified by two pointers: the first one points to the *first*
element to process (``begin``), the second one points to the *next after the last* element to process. So why have
we choosen an open range ``[begin; end)``? This will allow us to write down the loop end condtion in a very clear
and obvious way.

The implementation is given below:

.. code-block:: cpp

    template<class T>
    const T* max_element(const T* begin, const T* end)
    {
        const T* max = begin;

        for (++begin; begin != end; ++begin)
        {
            if (*begin > * max)
                max = begin;
        }

        return max;
    }

Using this implementation one can find the maximum element both in the whole array and in any part of it:

.. code-block:: cpp

    int array[] = {1, 2, 3, 4, 5, 6};

    int max = max_element(array, array + 6);
    int max_of_first3 = max_element(array, array + 3);



Can we do even better?
----------------------

Our generalized function works great for arrays of any type, but arrays have a few drawbacks:
one can't resize an existing array, one can't push a new element to the front of an array, etc.
It would be great to use the same code for searching of the maximum element in different kinds of
data structures, e. g. in a `linked list <http://en.wikipedia.org/wiki/Linked_list>`_.

But elements of a linked list aren't allocated linearly in memory - they are separate chunks of
memory connected with each other using pointers (*links*). That means we can't use
our function anymore, because the way we access data structure elements has changed, though
the algorithm itself remains completely the same: visit all the elements one after another and
compare each one with the current maximum element.

What really has changed is the way we access data structure elements. For arrays we could use
pointers arithmetic to calculate the address of an element we want (``base address + index * sizeof(T)``).
But to access an ``i-th`` element of a linked list one should go from the ``head`` of the list
to the ``(i-1)-th`` element one by one using pointers stored in the list nodes.

So here is the problem with our function: **the way we access elements of used data structure
is tightly coupled with the algorithm which is implemented by our function**. That means
we have to write a separate version of our function for all data structures we want to use
it for. And we all know that code duplication is a really bad thing which leads to errors and
difficults during the process of refactoring.

To solve this problem we have to decouple the algorithm from a data structure it processes.

Lets have a look at our function:

.. code-block:: cpp

    template<class T>
    const T* max_element(const T* begin, const T* end)
    {
        const T* max = begin;

        for (++begin; begin != end; ++begin)
        {
            if (*begin > * max)
                max = begin;
        }

        return max;
    }

How do we access the array elements?

1. ``*`` – *dereference a pointer* – get the value of an element the pointer points to.
2. ``!=`` – *not equal* – compare two pointers (to detect the end of a range).
3. ``++`` – *increment a pointer* – move the pointer to the next element of an array.

If we pass some objects to the ``max_element()`` function instead of passing pointers,
we can define the operations given above for these objects in a way they implement the
logic of accessing elements of different data structures (e. g. a linked list).

It is easy to do using templates and operators overloading facilities of ``C++``.

So the final version of our function looks like this:

.. code-block:: cpp

    template<class Iterator>
    Iterator max_element(Iterator begin, Iterator end)
    {
        Iterator max = begin;

        for (++begin; begin != end; ++begin)
        {
            if (*begin > * max)
                max = begin;
        }

        return max;
    }

So here we use objects of template parameter class ``Iterator`` instead of pointers. So what's an
iterator?

An **iterator** is a special object which allows one to access a data structure elements without
exposing its internal implementation. One works with a data structure by the means of a well defined
abstract interface of iterators.

``C++`` iterators use pointers semantics as their interface, but that's just an
implementation detail. It is important that all containers provide iterators with the same interface.
Users work with containers using iterators and don't know anything about how the containers are
actually implemented. This is the way algorithms and data structures are decoupled - one can use
the same algorithm for different data structures without any changes of the code.

Your first iterator
-------------------

Consider the simplest implementation of a singly linked list. The implementation of a list node:

.. code-block:: cpp

    #ifndef __LIST_NODE_H__
    #define __LIST_NODE_H__

    template<class T>
    struct Node
    {
        T data;
        Node<T>* next;
    };

    #endif /* __LIST_NODE_H__ */


The implementation of a list container:

.. code-block:: cpp

    #ifndef __LINKED_LIST_H__
    #define __LINKED_LIST_H__

    #include "list_node.h"
    #include "list_iterator.h"

    template<class T>
    class LinkedList
    {
    public:
        LinkedList();
        ~LinkedList();

        ListIterator<T> begin() const;
        ListIterator<T> end() const;

        void push_front(const T& elem);
        void push_back(const T& elem);

    private:
        Node<T>* _head;
        Node<T>* _tail;
    };

    template<class T>
    LinkedList<T>::LinkedList()
        : _head(0), _tail(0)
    { }

    template<class T>
    LinkedList<T>::~LinkedList()
    {
        while (_head)
        {
            Node<T>* next = _head->next;
            delete _head;
            _head = next;
        }
    }

    template<class T>
    void LinkedList<T>::push_front(const T& elem)
    {
        if (!_head)
        {
            _head = new Node<T>;
            _head->data = elem;
            _head->next = 0;

            _tail = _head;
        }
        else
        {
            Node<T>* oldfirst = _head;

            _head = new Node<T>;
            _head->data = elem;
            _head->next = oldfirst;
        }
    }

    template<class T>
    void LinkedList<T>::push_back(const T& elem)
    {
        if (!_tail)
        {
            _tail = new Node<T>;
            _tail->data = elem;
            _tail->next = 0;

            _head = _tail;
        }
        else
        {
            Node<T>* oldlast = _tail;

            _tail = new Node<T>;
            _tail->data = elem;
            _tail->next = 0;

            oldlast->next = _tail;
        }
    }

    template<class T>
    ListIterator<T> LinkedList<T>::begin() const
    {
        return ListIterator<T>(_head);
    }

    template<class T>
    ListIterator<T> LinkedList<T>::end() const
    {
        return ListIterator<T>(0);
    }

    #endif /* __LINKED_LIST_H__ */


We have implemented a minimal subset of list methods:

* data structure allocation/initialization and destruction/deallocation - ``LinkedList()``, ``~LinkedList()``;
* adding of elements to the front and to the back of a list - ``push_front()``, ``push_back()``;
* access to a list elements - methods that return iterators which point to the begin and to the
  end of the list - ``begin()``, ``end()``.

Using the iterators that are returned by ``begin()/end()`` pair one can access all the list elements.
The implementation of an iterator for a linked list data structure is given below:

.. code-block:: cpp

    #ifndef __LIST_ITERATOR_H__
    #define __LIST_ITERATOR_H__

    template<class T>
    class ListIterator
    {
    public:
        ListIterator(Node<T>* node);

        const Node<T>* node() const;

        ListIterator<T>& operator++();
        const T& operator*() const;
        bool operator!=(const ListIterator<T>& it) const;

    private:
        Node<T>* _currentNode;
    };

    template<class T>
    ListIterator<T>::ListIterator(Node<T>* node)
        : _currentNode(node)
    { }

    template<class T>
    const Node<T>* ListIterator<T>::node() const
    {
        return _currentNode;
    }

    template<class T>
    ListIterator<T>& ListIterator<T>::operator++()
    {
        _currentNode = _currentNode->next;
        return *this;
    }

    template<class T>
    const T& ListIterator<T>::operator*() const
    {
        return _currentNode->data;
    }

    template<class T>
    bool ListIterator<T>::operator!=(const ListIterator<T>& it) const
    {
        return _currentNode != it.node();
    }

    #endif /* __LIST_ITERATOR_H__ */

An iterator instance is initialized with a pointer to a linked list node. Overloaded operators implement
the logic of accessing list nodes and values they contain.

Lets have a look at how our function for returning of an iterator which points to the maximum element of
a given data structure works for both arrays and linked lists:

.. code-block:: cpp

    LinkedList<int> l;
    l.push_front(1);
    l.push_back(2);
    l.push_back(3);
    l.push_back(10);
    l.push_back(4);
    l.push_front(5);

    int arr[] = {1, 2, 3, 10, 4, 5};

    std::cout << "Max in list: "  << *max_element(l.begin(), l.end()) << " ";
    std::cout << "Max in array: " << *max_element(arr, arr + 6) << " ";


Conclusion
----------

This is only a small part of what you need to know about iterators. We've covered only one kind of iterators
- **Forward Iterators** which allows one to access elements moving forwards (using the ``++`` operator). There
are other kinds of iterators, e. g. **Bidirectional Iterators** which allows one to access elements moving
backwards too (``++``, ``!=``, ``*`` operators are suplemented with ``--`` operator), etc.

``C++`` iterators use pointers semantics as their interface, but it's just an
implementation detail - any other interface could have been choosen. But the actual
implementation enables one to use raw pointers as iterators.

Iterators are a very important part of ``STL`` because they decouple algorithms from data structures
these algorithms work on. This way algorithms might be generalized to work with any data structure
as long as it provides the required kinds of iterators.

The source code of code snippets is available on `GitHub <https://github.com/malor/iterators-source>`_.
