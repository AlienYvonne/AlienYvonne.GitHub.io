

## Lua内存存储方式

1. 字符串

   1. 全局字符串 stringtable

2. TValue

   1. 基本类型
   2. GCObject

3. table

   1. 结构

      ```c
      typedef struct Table {
        CommonHeader; // #define CommonHeader	struct GCObject *next; lu_byte tt; lu_byte marked
        lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
        lu_byte lsizenode;  /* log2 of size of 'node' array */
        unsigned int alimit;  /* "limit" of 'array' array */
        TValue *array;  /* array part */
        Node *node;
        Node *lastfree;  /* any free position is before this position */
        struct Table *metatable;
        GCObject *gclist;
      } Table; // lua 32位虚拟机
      
      typedef struct Node {
        TValue i_val;
        TKey i_key;
      } Node;
      
      typedef union TKey {
        struct {
          TValuefields;
          struct Node *next;  /* for chaining */
        } nk;
        TValue tvk;
      } TKey; // union 共用体 占用内存为最长的成员占用的内存
      
      
      ```

   2. table基础结构大小 

      >t = {}
      >
      >table size:    64    bytes (项目内Lua)（？是因为64位+项目内修改吗？
      >
      >table size:     32   bytes (Lua 5.1.4  32位
      >
      >table size:   	32 bytes(Lua 5.4.2 32位
      >
      >table size: 	56 bytes(lua 5.4.2 64位

   3. 内存分配

      1. Array Resize的时候  

         1. 5.4 最大整数Key为n   预分配大于n的第一个2的幂方数 
         2. 5.1 当前内存size翻倍

      2. Rehash 对数组内存寻找满足条件的最优size

         1. ```c
            /*
            ** {=============================================================
            ** Rehash
            ** ==============================================================
            */
            
            /*
            ** Compute the optimal size for the array part of table 't'. 'nums' is a
            ** "count array" where 'nums[i]' is the number of integers in the table
            ** between 2^(i - 1) + 1 and 2^i. 'pna' enters with the total number of
            ** integer keys in the table and leaves with the number of keys that
            ** will go to the array part; return the optimal size.  (The condition
            ** 'twotoi > 0' in the for loop stops the loop if 'twotoi' overflows.)
            */
            static unsigned int computesizes (unsigned int nums[], unsigned int *pna) {
              int i;
              unsigned int twotoi;  /* 2^i (candidate for optimal size) */
              unsigned int a = 0;  /* number of elements smaller than 2^i */
              unsigned int na = 0;  /* number of elements to go to array part */
              unsigned int optimal = 0;  /* optimal size for array part */
              /* loop while keys can fill more than half of total size */
              for (i = 0, twotoi = 1;
                   twotoi > 0 && *pna > twotoi / 2;
                   i++, twotoi *= 2) {
                a += nums[i];
                if (a > twotoi/2) {  /* more than half elements present? */
                  optimal = twotoi;  /* optimal size (till now) */
                  na = a;  /* all elements up to 'optimal' will go to array part */
                }
              }
              lua_assert((optimal == 0 || optimal / 2 < na) && na <= optimal);
              *pna = na;
              return optimal;
            }
            
            ```

      3. 处理hash冲突方式：empty position 使用空闲节点

         ```c
         /*
         ** inserts a new key into a hash table; first, check whether key's main 
         ** position is free. If not, check whether colliding node is in its main 
         ** position or not: if it is not, move colliding node to an empty place and 
         ** put new key in its main position; otherwise (colliding node is in its main 
         ** position), new key goes to an empty position. 
         */
         static TValue *newkey (lua_State *L, Table *t, const TValue *key) {
           Node *mp = mainposition(t, key);
           if (!ttisnil(gval(mp)) || mp == dummynode) {
             Node *othern;
             Node *n = getfreepos(t);  /* get a free place */
             if (n == NULL) {  /* cannot find a free place? */
               rehash(L, t, key);  /* grow table */
               return luaH_set(L, t, key);  /* re-insert key into grown table */
             }
             lua_assert(n != dummynode);
             othern = mainposition(t, key2tval(mp));
             if (othern != mp) {  /* is colliding node out of its main position? */
               /* yes; move colliding node into free position */
               while (gnext(othern) != mp) othern = gnext(othern);  /* find previous */
               gnext(othern) = n;  /* redo the chain with `n' in place of `mp' */
               *n = *mp;  /* copy colliding node into free pos. (mp->next also goes) */
               gnext(mp) = NULL;  /* now `mp' is free */
               setnilvalue(gval(mp));
             }
             else {  /* colliding node is in its own main position */
               /* new node will go into free position */
               gnext(n) = gnext(mp);  /* chain new position */
               gnext(mp) = n;
               mp = n;
             }
           }
           gkey(mp)->value = key->value; gkey(mp)->tt = key->tt;
           luaC_barriert(L, t, key);
           lua_assert(ttisnil(gval(mp)));
           return gval(mp);
         }
         ```

         

      4. 链表保存

   4. 数组内存和hash内存连续

   

   



