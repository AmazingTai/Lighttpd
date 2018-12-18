USET是一个宏类

```
struct data_unset *(*copy)(const struct data_unset *src); 
void (* free)(struct data_unset *p); 
void (* reset)(struct data_unset *p); 
int (*insert_dup)(struct data_unset *dst, struct data_unset *src); 
void (*print)(const struct data_unset *p, int depth)

```
五个指针成员。

```
typedef enum { 
        TYPE_UNSET,         /* 数据的类型未设置，
                               这几种数据类型使用了面向对象的设计思想，
                               这个类型相当于父类型 
                             */
        TYPE_STRING,         /* 字符串类型 */
        TYPE_COUNT,         /* COUNT类型 */
        TYPE_ARRAY,         /* 数组类型 */
        TYPE_INTEGER,     /* 整数类型 */
        TYPE_FASTCGI,     /* FASTCGI类型 */
        TYPE_CONFIG         /* CONFIG类型 */
} data_type_t;

```
通用数组类型在array.h中声明。

```
typedef struct 
{
    /* UNSET类型的指针型数组，存放数组中的元素 */
    data_unset **data;
    /* 按 data 数据的排序顺序保存 data 的索引 */
    size_t *sorted;
    size_t used;    /* data中已经使用了的长度，也就是数组中元素个数 */

        /* data的大小。data的大小会根据数据的多少变化，会为以后的数据预先分配空间 */
    size_t size;    
    /* 用于保存唯一索引,初始为 0,之后递增 */
    size_t unique_ndx;
    /* 比used大的最小的2的倍数。也就是离used最近的且比used大的2的倍数 ，用于在数组中利用二分法查找元素*/
    size_t next_power_of_2;
    /* data is weakref, don't bother the data */
    /* data就是一个指针，不用关系其所指向的内容 */
    int is_weakref;                
} array;

```
通用数组结构体定义



1、array *array_init(void); 
初始化数组，分配空间。

2、array array_init_array(array a); 
用数组a来初始化一个数组。也就是得到一个a的深拷贝。

3、void array_free(array * a); 
释放数组。释放所有空间。

4、void array_reset(array * a); 
重置data中的所有数据（调用UNSET类型数据中的reset函数），并将used设为0。相当于清空数组。

5、int array_insert_unique(array * a, data_unset * str); 
将str插入到数组中，如果数组中存在key与str相同的数据，则把str的内容拷贝到这个数据中。

6、data_unset array_pop(array a); 
弹出data中的最后一个元素，返回其指针，data中的最后一个位置设为NULL。

7、int array_print(array * a, int depth); 
打印数组中的内容。depth参数用于在打印多维数组时，实现缩进。

8、a_unset array_get_unused_element(array a, data_type_t t); 
返回第一个未使用的数据，也就是used位置的数据，这个数据不在数组中，返回这个数据指针后，将data[unsed]设为NULL。可能返回NULL。

9、data_unset array_get_element(array a, const char *key); 
根据key值，返回数组中key值与之相同的数据

10、data_unset array_replace(array a, data_unset * du); 
如果数组中有与du的key值相同的数据，则用du替换那个数据，并返回那个数据的指针。如果不存在，则把du插入到数组中。（调用data_insert_unique函数）

11、 int array_strcasecmp(const char *a, size_t a_len, const char *b, size_t b_len); 
这个函数并没实现，仅仅给出了上面的定义。

12、void array_print_indent(int depth); 
根据depth打印空白，实现缩进。

13、size_t array_get_max_key_length(array * a); 
返回数组中最长的key的长度。

