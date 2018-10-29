## Overview

The Trie tree, also known as the dictionary tree, word search tree or prefix tree, is a multi-fork tree structure for fast retrieval. For example, the dictionary tree of English letters is a 26-fork tree, and the digital dictionary tree is a 10-fork tree.

The word Trie comes from re trie ve, pronounced /tri:/ "tree", and some people read /traɪ/ "try".

Trie trees can take advantage of the common prefix of strings to save storage space. As shown in the following figure, the trie tree saves 6 strings tea, ten, to, in, inn, int with 10 nodes:

![trie](picture/trietree.jpg)

In the trie tree, the common prefix for the strings in, inn, and int is "in", so you can store only one copy of "in" to save space. Of course, if there are a large number of strings in the system and these strings have no common prefix, the corresponding trie tree will consume a lot of memory, which is also a disadvantage of the trie tree.

The basic properties of the Trie tree can be summarized as:

1. The root node does not contain characters, except for the root node, each node contains only one character.

1. From the root node to a node, the characters passing through the path are connected, which is the string corresponding to the node.

1. All the children of each node contain different strings.

## Operations

The Insert, Delete, and Find of the letter tree are very simple. You can use a one-cycle loop, that is, the i-th loop finds the subtree corresponding to the first i letters, and then performs the corresponding operations. To implement this letter tree, we can save it with the most common array (static memory), and of course we can also open dynamic pointer types (dynamically open memory). As for the point of the node to the son, there are generally three ways:

1. Open an array of letter set size for each node, the corresponding subscript is the letter represented by the son, and the content is the position of the son corresponding to the large array, that is, the label;

2. Hang a linked list for each node and record who each son is in a certain order;

3. Record the tree using the left son and the right brother.

The three methods have their own characteristics. The first type is easy to implement, but the actual space requirements are relatively large; the second type is easier to implement, the space requirement is relatively small, but it is more time consuming; the third type, the space requirement is the smallest, but it is relatively time consuming and difficult to write.

The following shows the implementation of dynamic memory development:

```C
#define MAX_NUM 26
enum NODE_TYPE{ //"COMPLETED" means a string is generated so far.
  COMPLETED,
  UNCOMPLETED
};
struct Node {
  enum NODE_TYPE type;
  char ch;
  struct Node* child[MAX_NUM]; //26-tree->a, b ,c, .....z
};

struct Node* ROOT; //tree root

struct Node* createNewNode(char ch){
  // create a new node
  struct Node *new_node = (struct Node*)malloc(sizeof(struct Node));
  new_node->ch = ch;
  new_node->type == UNCOMPLETED;
  int i;
  for(i = 0; i < MAX_NUM; i++)
    new_node->child[i] = NULL;
  return new_node;
}

void initialization() {
//intiazation: creat an empty tree, with only a ROOT
ROOT = createNewNode(' ');
}

int charToindex(char ch) { //a "char" maps to an index<br>
return ch - 'a';
}

int find(const char chars[], int len) {
  struct Node* ptr = ROOT;
  int i = 0;
  while(i < len) {
   if(ptr->child[charToindex(chars[i])] == NULL) {
   break;
  }
  ptr = ptr->child[charToindex(chars[i])];
  i++;
  }
  return (i == len) && (ptr->type == COMPLETED);
}

void insert(const char chars[], int len) {
  struct Node* ptr = ROOT;
  int i;
  for(i = 0; i < len; i++) {
   if(ptr->child[charToindex(chars[i])] == NULL) {
    ptr->child[charToindex(chars[i])] = createNewNode(chars[i]);
  }
  ptr = ptr->child[charToindex(chars[i])];
}
  ptr->type = COMPLETED;
}
```

## Advanced implementation

Can be implemented in a double array (Double-Array). The use of double arrays can greatly reduce memory usage

![double](picture/double-array-trie.jp)

## Usecases

Trie is a very simple and efficient data structure, but there are a large number of application examples.

(1) String retrieval

Save some information about the known strings (dictionaries) in the trie tree in advance to find out if other unknown strings have appeared or appeared frequently.

Example:

@ Give a list of vocabulary words consisting of N words, and an article written in lowercase English. Please write all the new words that are not in the vocabulary list in the earliest order.

@ Give a dictionary where the words are bad words. Words are all lowercase letters. A piece of text is given, and each line of text is also composed of lowercase letters. Determine if the text contains any bad words. For example, if rob is a bad word, the text problem contains bad words.

(2) The longest common prefix of the string

The Trie tree uses the common prefix of multiple strings to save storage space. Conversely, when we store a large number of strings on a trie tree, we can quickly get the common prefix of some strings.

Example:

@ Give N lowercase English alphabet strings, and Q queries, which is the length of the longest common prefix for asking two strings?

Solution: First create a corresponding letter tree for all strings. At this point, it is found that the length of the longest common prefix for the two strings is the number of common ancestors of the node at which they are located, so the problem translates into the nearest Recent Common Ancestor (LCA) problem.

The recent public ancestor problem is also a classic problem, which can be done in the following ways:

1. Using the Disjoint Set, you can use the classic Tarjan algorithm;

2. After finding the Euler Sequence of the letter tree, you can turn it into the classic Minimum Minimum Query (RMQ) problem.

(About and check, Tarjan algorithm, RMQ problem, there is a lot of information online.)

(3) Sorting

The Trie tree is a multi-fork tree. As long as the whole tree is pre-ordered, the corresponding string is output in lexicographic order.

Example:

@ Gives you N different English names consisting of only one word, letting you sort them out lexicographically from small to large.

(4) As an auxiliary structure of other data structures and algorithms

Such as suffix tree, AC automaton, etc.

## Trie tree complexity analysis

(1) The time complexity of insertion and lookup is O(N), where N is the length of the string.

(2) The space complexity is 26^n level, which is very large (can be improved by double array implementation).

## Summary

Trie tree is a very important data structure. It has a wide range of applications in information retrieval, string matching, etc. At the same time, it is also the basis of many algorithms and complex data structures, such as suffix trees, AC automata, etc. Therefore, Mastering the data structure of Trie Tree is very basic and necessary for an IT staff!
