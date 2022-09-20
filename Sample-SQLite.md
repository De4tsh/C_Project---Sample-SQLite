# SQLite结构

![](https://raw.githubusercontent.com/De4tsh/typoraPhoto/main/img/202209101005737.jpg)

SQLite由三个子系统中的8个独立模块组成

-   **接口（Interface）**

    接口由SQLite C API组成，通过该接口完成于SQLite的交互

-   **编译器（Compiler）**

    由三部分构成，首先分词器（Tokenizer）、分析器（Parser）协作处理文本形式的结构化查询（也就是我们输入的SQL语句）语句，分析其语法有效性，转化为底层更方便处理的数据结构---语法树，然后再将语法树传给代码生成器（Code
    generator）进行处理

    代码生成器生成一种SQLite专用的代码，最后由虚拟机（Virtual
    Machine）进行执行

    -   **分词器（Tokenizer）**

    -   **分析器（Parser）**

    -   **代码生成器（Code generator）**

-   **虚拟机（Virtual Machine）**

    主要进行数据库操作，它的每一条指令或者用来完成特定的数据库操作(比如打开一个表的游标、开始一个事务等)，或者为完成这些操作做准备。总之，所有的这些指令都是为了满足SQL命令的要求

    它可以对一个或多个表或索引执行操作，每个表或索引都存储在称为B-Tree的数据结构中

-   **后端（Back-end）**

    B-tree和page
    cache共同对数据进行管理。它们操作的是数据库页，这些页具有相同的大小，就像集装箱。页里面的"货物"是表示信息的大量bit，这些信息包括记录、字段和索引入口等。B-tree和pager都不知道信息的具体内容，它们只负责"运输"这些页，页不关心这些"集装箱"里面是什么

    B-tree的主要功能就是索引，它维护着各个页之间的复杂的关系，B-tree由许多节点组成，每个节点的长度为一页，便于快速找到所需数据。它把页组织成树型的结构(这是它名称的由来)，这种树是为查询而高度优化了的。

    Page为B-tree服务，为它提供页。**Pager的主要作用就是通过OS接口在B-tree和磁盘之间传递页**。磁盘操作是计算机到目前为止所必须做的最慢的事情。所以，pager
    尽力提高速度，其方法是把经常使用的页存放到内存当中的页缓冲区里，从而尽量减少操作磁盘的次数。它使用特殊的算法来预测下面要使用哪些页，从而使B-tree能够更快地工作

    -   **B-Tree**

    -   **页缓冲（Page Cache/Pager）**

    -   **操作系统接口（系统调用）**

# 实现

## 第一部分：REPL（read-execute-print） {#第一部分replread-execute-print）}

当从命令行启动SQLite时，他都会启动一个REPL的界面，**由于循环接受用户的输入**

![](https://raw.githubusercontent.com/De4tsh/typoraPhoto/main/img/202209101037171.png)

为此我们在`main`函数中实现该功能

此时我们要实现的功能，除了要循环的显示`sqlite>`这种提示外，还要接收用户的输入，并将其保存在内存中，用于接下来对该语句含义的分析与执行

首先先完成一个简单的函数：打印`sqlite>`

``` {.c}
void print_prompt()
{
    printf("sqlite>"); // 配合while(1)即可完成循环输出
}
```

然后我们接下来需要使用`getline()`函数接受用户的输入，该函数的原型为

``` {.c++}
ssize_t getline(char **lineptr,size_t *n,FILE *stream)
    // lineptr 指向我们用来指向包含读取行的缓冲区的变量的指针
    // n 指向我们用来保存分配缓冲区大小的变量的指针
    // stream 要读取的输入流，我们从标准输入流读取即可
    // return value 读取的字节数，可能小于缓冲区的大小
```

所以为了满足上述参数，我们需要**定义一个结构体，用于存取用户输入语句的各项信息**

注意定义值的类型要与getline各项参数/返回值的类型相同（脱一层指针后的类型）

``` {.c}
typedef struct
{
    char *buffer; // lineptr
    size_t buffer_length; // n
    ssize_t input_length; // return value
    
        /*
        ssize_t 有符号
        size_t 无符号
	    */
    
}InputBuffer; // 用于保存用户输入的各项信息
```

定义完了结构体后，我们就需要**接受用户的输入并分配空间**

1.  初始化`InpurBuffer`

    ``` {.c}
    InputBuffer* new_input_buffer()
    {
    	InputBuffer* input_buffer = (InputBuffer *)malloc(sizeof(InputBuffer));
        
        input_buffer->buffer = NULL;
        input_buffer->buffer_length = 0;
        input_buffer->input_length = 0;
        
        return input_buffer;
    }
    ```

2.  读取用户输入，并为其输入的字符串分配空间

    ``` {.c}
    void read_input(InputBuffer* input_buffer)
    {
        
        ssize_t bytes_read = getline(&(input_buffer->buffer),&(input_buffer->buffer_length),stdin);
        
        // 此处由于我们的input_buffer->buffer = NULL 所以getline会为存入的字符串malloc空间
        // 该空间的释放由用户完成
    	
        // 若未读取到字符串或读取失败
        if (bytes_read <= 0)
        {
    		printf("Error reading input\n");
             exit(EXIT_FAILUR);
        }
        
        // 由于我们输入完数据需要点回车，属于分隔符不属于数据，所以此处需要减掉一个
        input_buffer->input_length = bytes_read - 1;
        // 将最后的分隔符替换为字符串的结尾 \0
        input_buffer->buffer[bytes_read -1] = 0;

    }
    ```

**既然分配了空间，就肯定要释放**，这里先写一个简单的释放的函数，随着后面功能的加多，会重写该函数

``` {.c}
void close_input_buffer(InputBuffer* input_buffer)
{
    free(input_buffer->buffer); // getline malloc 用户 free
    free(input_buffer);
}
```

既然写了释放的函数，那么释放的时机也必须得确认出来，这里我们先提供一条元命令`.exit`用于退出程序释放空间

``` {.c}
if (strcmp(input_buffer->buffer,".exit") == 0) // 若输入.exit
{
	close_input_buffer(input_buffer); // 释放空间
     exit(EXIT_SUCCESS);
}
else
{
    printf("Unrecongnized command '%s'.\n",input_buffer->buffer);
    close_input_buffer(input_buffer);
}
```

### 最终在main函数中完成调用即可

``` {.c}
#include <stdbool.h>
/*	
	该库文件于C99中加入，stdbool头文件中顶一个4个宏，定义出了布尔类型

	bool：定义为_Bool。
	true：定义为1。
	false：定义为0。
	上述三个有定义的前提是不是C++的代码
	__bool_true_false_are_defined：定义为1。
*/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int _tmain(int argc,TCHAR* argv[])
{
	InputBuffer* input_buffer = new_input_buffer();
    
    while(true)
    {
        print_prompt();
        read_input(input_buffer);
        
        if (strcmp(input_buffer->buffer,".exit") == 0) // 若输入.exit
		{
			 close_input_buffer(input_buffer); // 释放空间
     		 exit(EXIT_SUCCESS);
		}
		else
		{
    		printf("Unrecongnized command '%s'.\n",input_buffer->buffer);
    		close_input_buffer(input_buffer);
		} 
	}
}
```

# 第二部分 - World\'s Simplest SQL Compiler and Virtual Machine

第二部分我们将简单的完成SQL编译器（`SQL Compiler`）与最简单的虚拟机（`Virtual Machine`），**SQL编译器会解析我们的SQL语句将其转化为虚拟机可以识别的指令（解析字符串并输出称为字节码的内部表示）并被虚拟机执行**，但我们当前要实现的要简单的多，在后续的部分中还会对该部分代码进行重构，别急

我们数据库中执行的命令可以分为这样两种：

1.  **元命令**

    元命令可以方便我们更快捷的管理数据库，比如说最简单的退出指令就是一个元命令，用于退出数据库

    元命令统一以 `.` 开头 比如
    `.exit`，因此我们可以凭借该指纹辨识出他们，并在单独的函数中处理他们

2.  **SQL语句**

    对数据库进行操作的语句

## 元命令的定义及实现

对于元命令我们无需对其执行分词、分析等操作，直接识别执行即可，为了标识元命令是否被执行成功，我们定义枚举来表示其状态

``` {.c}
// 元命令是否被虚拟机执行成功
typedef enum
{
    META_COMMAND_SUCCESS;
    META_COMMAND_UNRECOGNIZED_COMMAND
}MetaCommandResult;
```

之后我们通过专门的函数实现其功能即可

``` {.c}
MetaCommandResult do_meta_command(InputBuffer* input_buffer)
{
    if (strcmp(input_buffer->buffer,".exit") == 0)
    {
        exit(EXIT_SUCCESS); // 退出数据库
    }
    else
    {
        return MEAT_COMMAND_UNRECONGNIZED_COMMAND; // 元命令无法识别
    }
}
```

## SQL语句的编译与执行

前面说到对于SQL语句我们需要**先通过内置的编译器将其编译为可以内部虚拟机可以识别的指令再被内部的虚拟机执行**，所以第一步，为了标识该命令：

-   是否**被SQL编译器编译成功**

-   是否**被虚拟机执行成功**（该枚举状态的定义会在下一部分实现）

我们需要通过枚举将其状态定义出来：（同样在后续该代码会被重构出更多状态）

``` {.c}
// SQL语句是否被SQL编译器编译成功
typedef enum
{
    PREPARE_SUCCESS,
    PREPARE_UNRECONGNIZED_STATEMENT，
    PREPARE_SYNTAX_ERROR // 第三部分添加，用于标识SQL语句语法错误
}PrepareResult;

// 编译器识别出语句的类型
typedef enum
{
    STATEMENT_INSERT,
    STATEMENT_SELECT
        
}StatementType
    
// 用结构体将上述类型进行封装    
// 因为后需要先通过SQL编译器进行识别编译，若是别成功再传递给虚拟机执行，识别的结果
// 就是通过该结构体传递的
typedef struct
{
    StatementType type;
    
}Statement;
```

### SQL编译器框架实现（SQL Complier） {#sql编译器框架实现sql-complier）}

``` {.c}
PrepareResult prepare_statement(InputBuffer* input_buffer,Statement* statement)
{
    // 若为insert命令
    /*
    	由于insert命令一般后接插入的数据，所以此处对于insert的识别
    	我们仅截取前6位即可
    */
    if (strcmp(input_buffer->buffer,"insert",6) == 0)
    {
        statement->type = STATEMENT_INSERT;
        return PREPARE_SUCCESS;
    }
    
    // 若为select命令
    /*
    	对于select命令，我们数值的select也是带要查询的数据的，这些我们在后续
    	进行重构，此处的select不需接任何参数，会返回本数据库中的所有数据
    */
    if (strcmp(input_buffer->buffer,"select") == 0)
    {
        statement->type = STATEMENT_SELECT;
        return PREPARE_SUCCESS;
    }
    
    // 当前我们仅支持insert和select，若不是他俩，就返回无法识别
    return PREPARE_UNRECONGNIZED_STATEMENT;
}
```

### 虚拟机执行（Virtual Machine） {#虚拟机执行virtual-machine）}

本部分我们仅定义出虚拟机应有的框架，具体insert与select的实现，以及数据存储的格式等具体的内容，将在下部分实现

``` {.c}
void execute_statement(Statement* statement)
{
    swicth (statement->type)
    {
        case (STATEMENT_INSERT):
        	printf("insert.\n");
        	break;
        case (STATEMENT_SELECT):
        	printf("select.\n");
        	break;
    }
}
```

## 重构main()函数

第一部分中我们实现的main()函数中仅能识别出`.exit`，而且有一个明显的不足是`.exit`的实现实在`main()`函数中进行的，为了增加其重用性，我们将该模块分离出来，并引入对SQL语句的识别等的调用

第一部分的main函数：

``` {.c}
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int _tmain(int argc,TCHAR* argv[])
{
	InputBuffer* input_buffer = new_input_buffer();
    
    while(true)
    {
        print_prompt();
        read_input(input_buffer);
        
        if (strcmp(input_buffer->buffer,".exit") == 0) // 若输入.exit
		{
			 close_input_buffer(input_buffer); // 释放空间
     		  exit(EXIT_SUCCESS);
		}
		else
		{
    		printf("Unrecongnized command '%s'.\n",input_buffer->buffer);
    		close_input_buffer(input_buffer);
		} 
	}
}
```

**第二部分重构完的`main()`函数**

``` {.c}
int _tmain(int argc,TCHAR* argv[])
{
    InputBuffer* input_buffer = new_input_buffer();
    while(true)
    {
        print_prompt();
        read_input(input_buffer);
        
        // 若为元命令
        if (input_buffer->buffer[0] == '.')
        {
            switch (do_meta_command(input_buffer))
            {
                case (META_COMMAND_SUCCESS):
                    continue;
                case (META_COMMAND_UNRECONGNIZED_COMMAND):
                    printf("Unrecongnized command '%s'\n",input_buffer->buffer);
                    continue;
            }
        }
        
        // 若不为元命令
        Statement statement;
        switch (prepare_statement(input_buffer,&statement))
        {
            // SQL编译器编译成功成功识别命令
            case (PREPARE_SUCCESS):
                break;
            case (PREPARE_UNRECONGNIZED_COMMAND):
                printf("Unrecongnized keyword at start of '%s'.\n",input_buffer->buffer);
                continue;
		}
        
        // 执行命令
        execute_statement(&statement);
        printf("Executed.\n");            
    }
}
```

# 第三部分：An In-Memory, Append-Only, Single-Table Database

在前一部分中，我们为两种SQL语句的解析与执行搭建了框架，那么这一部分我们将首次初步完善`insert`与`select`的功能

## Insert命令

该部分我们将先实现`insert`的一部分功能，而着重点放在**`insert`后的数据在内存中的存储形式**

-   insert后的数据仅驻留在内存中（下部分再将其写入磁盘）

-   支持单个硬编码表

我们的硬编码表将用来存储用户的信息，格式为：

  column     type
  ---------- --------------
  id         integer
  username   varchar(32)
  email      varchar(255)

基于这种格式，insert的语句应为：`insert 1 Deutsh dann@gmail.com`

基于这种结构我们定义结构体来实现他

``` {.c}
#define COLUMN_USERNAME_SIZE 32 // username最大长度
#define COLUMN_EMAIL_SIZE 255 // email最大长度

typedef struct
{
    uint32_t id;
    char username[COLUMN_USERNAME_SIZE];
    char email[COLUMN_EMAIL_SIZE];
    
}Row; // 一份用户信息的结构体
```

但是我们想其实关于`insert`语句中要插入数据的提取也属于SQL编译器负责的范畴，所以结构体`Statement`中要增加有关`insert`要插入信息的参数,所以我们重构一下`Statement`

``` {.c}
typedef struct
{
    StatementType type;
    Row row_to_insert; // 只有再insert情况下，才会使用该项
    
}Statement;
```

但是我们好像遗漏一个问题就是，在哪里去获得这些用户的数据呢？

根据SQLite的结构获取数据的任务应该交给SQL编译器来完成，所以我们就通过重构`prepare_statement`函数来完成即可

我们先将上部分的`prepare_statement`函数拿过来，增加`insert`获取参数的代码

``` {.c}
PrepareResult prepare_statement(InputBuffer* input_buffer,Statement* statement)
{
    // 若为insert命令
    /*
    	由于insert命令一般后接插入的数据，所以此处对于insert的识别
    	我们仅截取前6位即可
    */
    if (strcmp(input_buffer->buffer,"insert",6) == 0)
    {
        statement->type = STATEMENT_INSERT;
        
+		 int args_assigned = sscanf(input->buffer->buffer,"insert %d %s +%s",&(statement-+>row_to_insert.id),&(statement->row_to_insert.username),&(statement->row_to_insert.email))
+		 if (args_assigned < 3)
+       {
+            return PREPARE_SYNTAX_ERROR;
+        }
        
        return PREPARE_SUCCESS;
    }
    
    // 若为select命令
    /*
    	对于select命令，我们数值的select也是带要查询的数据的，这些我们在后续
    	进行重构，此处的select不需接任何参数，会返回本数据库中的所有数据
    */
    if (strcmp(input_buffer->buffer,"select") == 0)
    {
        statement->type = STATEMENT_SELECT;
        return PREPARE_SUCCESS;
    }
    
    // 当前我们仅支持insert和select，若不是他俩，就返回无法识别
    return PREPARE_UNRECONGNIZED_STATEMENT;
}
```

接下来，我们需要将`insert`的数据保存于内存当中，我们的计划是：

-   将行存储在称为页（`Pages`）的内存块中

-   每个页要尽可能的存储多个`Row`

-   `Row`被序列化为每个页面（`Pages`）的紧凑表示

-   根据需求分配页（`Pages`）

-   建立一个指针数据用于指向已经分配的每一个页（`Pages`）

### Row数据在内存中的保存格式

> 紧凑表示

由于结构体内部对齐值的不同等因素会导致**结构体整体的实际大小 \>
里面所有数据大小之和**，所以我们要将结构体中的每个数据挨个拿出来保存起来，而不是直接开辟空间存储结构体

``` {.c}
// 此处是一个宏实现的表达式
// 此处一个很有意思的地方在于 sizeof((Struct *)0 -> Attribute)
// 这里是将 0 转化为一个该结构体的指针用来索引结构体中的参数，这样就可以省去定义一个结构体指针的空间
#define size_of_arrtibute(Struct,Attribute) sizeof(((Struct*)0->Attribute)

const uint32_t ID_SIZE = size_of_attribute(Row,id);
const uint32_t USERNAME_SIZE = size_of_attribute(Row,username);
const uint32_t EMAIL_SIZE = size_of_attribute(Row,email);

const uint32_t ID_OFFSET = 0;
const uint32_t USERNAME_OFFSET = ID_OFFSET + ID_SIZE;
const uint32_t EMAIL_OFFSET = USERNAME_OFFSET + USERNAME_SIZE;
const uint32_t ROW_SIZE = ID_SIZE + USERNAME_SIZE + EMAIL_SIZE;
```

然后我们按此结构将原输入的结构体中的数据拿出来序列化为该格式

``` {.c}
void serialize_row (Row* source,void* destination)
{
    memcpy(destination + ID_OFFSET,&(source->id),ID_SIZE);
    memcpy(destination + USERNAME_OFFSET,&(source->username),USERNAME_SIZE);
    memcpy(destination + EMAIL_OFFSET,&(source->email),EMAIL_SIZE);
}
```

但这种格式的数据要使用时，结构体是不认识他的，因为这是我们自定义的数据存储格式，**所以为了使用数据时可以从结构体中正常调用，我们还要定义饭序列化函数用于将数据恢复到结构体中**

``` {.c}
void deserialize_row (void* source,Row* destination)
{
    memcpy(&(destination->id),source + ID_OFFSET,ID_SIZE);
    memcpy(&(destination->username),source + USERNAME_OFFSET,USERNAME_SIZE);
    memcpy(&(destination->email),source + EMAIL_OFFSET,EMAIL_SIZE);
}
```

### 数据页存储

B-Tree用于索引，而B-Tree是以页为节点进行查找的，所以我们需要页这个结构，并将上述序列化后的数据存储于一个一个页中

**首先我们定义一些页的基本参数**

-   页的大小

-   最多支持多少页

-   每个页可以存放多少条Row数据

-   所有页一共可以存储多少条Row数据

``` {.c}
// 我们将一个页的大小设为4096个字节，也就是4KB大小
const uint32_t PAGE_SIZE = 4096; 

// 最多分配100个页用于存储数据库的数据
#define TABLE_MAX_PAGES 100 

// 每个页中可以存放多少条Row数据
// 防止Row数据越界（超过一个页的范围）
const uint32_t ROWS_PER_PAGES = PAGE_SIZE / ROW_SIZE; 

// 所有页加在一起一共可以放多少条Row数据
const uint32_t TABLE_MAX_ROWS = ROWS_PER_PAGE * TABLE_MAX_PAGES; 
```

最开始提到，为了方便的管理创建的每一个页，**需要建立一个指针数组指向每一个已经分配的页**

``` {.c}
typedef struct
{
	uint32_t num_rows;
    void *pages[TABLE_MAX_PAGES]; // 页指针数组
}Table;
```

有了这些页所必须的参数，我们就可以着手来初始化创建一个页表（数据条数与页指针数组）了，当然还有与之配到的释放页与页表的函数

``` {.c}
// 初始化创建页的基本信息
Table* new_table()
{
    Table* table = (Table*)malloc(sizeof(Table));
    table->num_rows = 0; // 初始时，页中数据的条数为0
    
    // 初始化页指针数组
    // 由于最开始我们还有没创建任意一个页，所以此时页指针数组中都为NULL
    // 只有在需要向页中存入数据时，才会由存数据的函数来申请页空间，并将申请到的页指针加	   // 入到该页指针数组中
    for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++)
    {
        table->pages[i] = NULL;
    }
    
    return table;
}

void free_table(Table* table)
{
	for (int i = 0; table->pages[i]; i++)
    {
        // 释放页表（页指针数组）中的每一个页
        free(table->pages[i]);
	}
    
    // 释放页表
    free(table);
}
```

接下来最重要的部分来了，既然定义了数据在内存中的存储格式，以及分页的管理机制，**[那么如何确定在内存中读取/写入特定的数据的位置，就是我们下一个要关注的点了]{.ul}**

``` {.c}
// 当此处参数中传入的row_num值表示是当前已经存在的数据总条数时，该函数的作用是定位到新传入的一条数据应该写入的位置

// 当此处参数中传入的row_num值表示是第多少多少条时，该函数的作用是查找该条数据的位置

void* row_slot(Table* table,uint32_t row_num)

{

    // 确定要读取/写入数据的地方的页号

    /*

    	若传入的数据的条数小于一页中最多可容纳的数据条数

    	就说明该数据就在第一页中

    */

	uint32_t page_num = row_num / ROWS_PER_PAGE;

    

    // 确认了页号后，根据在页指针数组中找到该页对应的指针

    void* page = table -> pages[page_num]

        

   	if (page == NULL)

    {

        // 若要找的页不存在，我们就分配空间建立该页

        // 并将新创建的页的指针加入页指针数组

        // 将当前的指针指向该页

        page = table->pages[page_num] = malloc(PAGE_SIZE)

    }

    

    // 此处参数中要传入的row_num值表示的是当前已经存在的数据总条数

    // 数据总条数 % 每一页可容纳的数据数 = 当前页已经存在的数据条数

    uint32_t row_offset = row_num % ROWS_PER_PAGE;

    

    // 当前页已经存在的数据条数 * 每条数据的大小 = 要存新数据的起始字节偏移量

    uint32_t byte_offset = row_offset * ROW_SIZE;

    

    // 返回所要插入新数据的位置（偏移）

    // 或返回所查找数据的位置（偏移）

    return page + byte_offset;

}
```

## 从页中存取数据

那么我们上面既然提供了从页中读取、向页中存储数据的接口，接下来我们就可以用之来实现我们的`insert`了

这里我们先回想一下，在SQLite的体系结构中，是谁来负责执行由SQL编译器处理完的指令来着？没错，是虚拟机（Virtual
Machine），为了表示虚拟机执行的结果（成功与否），类比于我们为SQL编译器执行结果的成功与否定义的枚举一样，**也需要为虚拟机的执行结果定义一个表示当前执行状态的枚举**

``` {.c}
typedef enum
{
    EXECUTE_SUCCESS, // 执行成功
    EXECUTE_TABLE_FULL // 执行失败，页表已满（所有页已被使用）
        
}ExecuteResult;
```

上一部分里，我们为`insert`语句的执行定义了一个这样的框架

**`void execute_statement(Statement* statement)`**

该部分我们要重构完成它，由于我们刚刚定义了执行完成的状态，所以此处返回值需要发生变化，而且我们需要向页中存储位于结构体中的用户数据，所以参数中还应添加页表的指针，故我们将函数重构为

``` {.c}
-void execute_statement(Statement* statement)
+ExecuteResult execute_statement(Statement* statement,Table *table)
{
    switch (statement->type)
    {
        case (STATEMENT_INSERT):
            return execute_insert(statement,table);
        case (STATEMENT_SELECT):
            return execute_select(statement,table);
    }
}
```

之后我们在分别实现`insert`和`select`的函数

``` {.c}


+ExecuteResult execute_insert(Statement* statement,Table* table)

{

    // 若当前页表中所有的页都已满，则不能再插入了

    if (table->num_rows >= TABLE_MAX_ROWS)

    {

        return EXECUTE_TABLE_FULL;

    }

    

    // 提示：statement->row_to_insert中存放着用户要插入的详细数据

    Row* row_to_insert = &(statement->row_to_insert);

    

    // 将序列化的后的数据写入页中

    serialize_row(row_to_insert,row_slot(table,table->num_rows));

    

    table->nums_rows += 1; // 插入了一条数据后，总的数据条数+1

    

    return EXECUTE_SUCCESS;

}
```

这样简单的`insert`就实现了，按照这种模式，我们把`select`也一块完成了

``` {.c}
ExecuteResult execute_select(Statement* statement,Table* table)

{

    Row row;

    for (uint32_t i = 0; i < table->num_rows; i++)

    {

        // 将内存中序列化后的数据反序列化为结构体中的数据结构

        deserialize_row(row_slot(table,i),&row);

        print_row(&row);

    }

    

    return EXECUTE_SUCCESS;

}
```

## `main()`函数 {#main函数-1}

好了现在到了第三部分的最后一步了，将我们上述实现的功能合并进`main()`函数即可

``` {.c}
+ #include <stdint.h>



int main(int argc, char* argv[]) 

{

+  Table* table = new_table();

   InputBuffer* input_buffer = new_input_buffer();

   while (true) {

     print_prompt();

     read_input(input_buffer);



+    if (input_buffer->buffer[0] == '.') 

+	{

+     switch (do_meta_command(input_buffer, table)) 

+	 {

+        case (META_COMMAND_SUCCESS):

+          continue;

+        case (META_COMMAND_UNRECOGNIZED_COMMAND):

+          printf("Unrecognized command '%s'\n", input_buffer->buffer);

+          continue;

+      }

+    }

+

+   Statement statement;

+   switch (prepare_statement(input_buffer, &statement)) 

+	{

+      case (PREPARE_SUCCESS):

+        break;

+      case (PREPARE_SYNTAX_ERROR):

+		printf("Syntax error. Could not parse statement.\n");

+		continue;

+      case (PREPARE_UNRECOGNIZED_STATEMENT):

+        printf("Unrecognized keyword at start of '%s'.\n",input_buffer->buffer);

+        continue;

+    }

+

+    switch (execute_statement(&statement, table))

+	{

+		case (EXECUTE_SUCCESS):

+	    	printf("Executed.\n");

+	    	break;

+		case (EXECUTE_TABLE_FULL):

+	    	printf("Error: Table full.\n");

+	    	break;

     }

   }

 }
```

### 补充一点

看上述的`main()`函数我们可以看出，MetaCommand的功能也进一步进行了封装

``` {.c}
typedef enum {
  META_COMMAND_SUCCESS,
  META_COMMAND_UNRECOGNIZED_COMMAND
} MetaCommandResult;
```

并在`main()`函数中修改了其实现

``` {.c}
    if (input_buffer->buffer[0] == '.') 
	{
     switch (do_meta_command(input_buffer, table)) 
	 {
        case (META_COMMAND_SUCCESS):
          continue;
        case (META_COMMAND_UNRECOGNIZED_COMMAND):
          printf("Unrecognized command '%s'\n", input_buffer->buffer);
          continue;
      }
   }
```

但显然，这并不是该部分的重点

# 第四部分 - Our First Tests (and Bugs)

本节来解决一些上面程序中设计不严谨的地方，与可能出现的`BUG`

### 字符串长度分配不合理

``` {.c}
typedef struct
{
	uint32_t id;
    
    char username[COLUMN_USERNAME_SIZE];
    char email[COLUMN_EMAIL_SIZE];
}Row;
```

这是之前用户数据格式的结构体，其中问题出现在`username`和`email`的长度分配上，我们最初的设想是，`username`最大存储的长度为`COLUMN_USERNAME_SIZE`（也就是用户输入的用户名的最大长度），email项同理，但这里我们忽略了一点：

C字符串默认是以`'\0'`结尾的，那就是说如果用户输入的长度为`COLUMN_USERNAME_SIZE`或`COLUMN_EMAIL_SIZE`时，会导致字符串结尾的`'\0'`被覆盖，导致输出乱码

**解决：多分配一个空间即可**

``` {.c}
typedef struct
{
	uint32_t id;
    
    char username[COLUMN_USERNAME_SIZE + 1];
    char email[COLUMN_EMAIL_SIZE + 1];
}Row;
```

### scanf()函数的安全性

> 我们在接受分割insert语句中的参数时，使用了scanf函数，而众所周知，scanf函数时没有长度限制的，易引发缓冲区溢出

``` {.c}
if (strncmp(input_buffer->buffer,"insert",6) == 0)
{
	statement->type = STATEMENT_INSERT;
    
    int args_assigned = sscanf(input_buffer->buffer,"insert %d %s %s",&(statement->row_to_insert.id),&(statement->row_to_insert.username),&(statement->row_to_insert.email));
    
    if (args_assigned < 3)
    {
        return PREPARE_SYNTAX_ERROR;
    }

    return PREPARE_SUCCESS;

}
```

上述这段我们之前写的代码显然存在着安全问题，因为使用了`scanf()`对参数进行提取，为了防止产生溢出，我们换一个思路，**使用`stroke()`函数以空格为分隔符提取到每一个参数，而且我们将另定义一个函数专门负责处理分割字符串**

``` {.c}
// 该函数专门负责分割出insert中插入的数据
// 并将分割出的数据赋给 statement->row_to_insert.(id | username | email)
PrepareResult prepare_insert(InputBuffer* input_buffer,Statement* statement)
{
    statement->type = STATEMENT_INSERT;
    
    // char *__cdecl strtok(char *_String, const char *_Delimiter)
    // strtok中的静态变量nextoken指针，用以记录分割一次之后_String中下一个字符串的起始位置
    // 所以要想再次对该字符串进行分割的话，第一个参数传NULL即可
    /*
    	strtok()分割字符串的机制是每一次将分隔符的位置替换为'\0'来截断字符串（在原字符串的基础		    上）
        所以不可以传入const字符串常量
    */
    char* keyword = strtok(input->buffer, " ");
    char* id_string = strtok(NULL," ");
    char* username = strtok(NULL," ");
    char* email = strtok(NULL," ")
    
    // 若三条数据中有哪一个没有被获取到，就说明插入语句语法错误
    if (id_string == NULL || username = NULL || email == NULL)
    {
		return PREPARE_SYNTAX_ERROR;
    }
    
    // int atoi (const char * str);
    // 将const char*转换为整型数
    int id = atoi(id_string);
    if (id < 0) // ID必须>0
    {
        return PREPARE_NEGATIVE_ID;
    }
    
    // 验证输入是否越界
    if (strlen(username) > COLUNM_USERNAME_SIZE)
    {	
        return PREPARE_STRING_TOO_LONG;
	}
    
    if (strlen(email) > COLUMN_EMAIL_SIZE)
    {
        return PREPARE_STRING_TOO_LONG;
    }
    
    // 若输入格式正确长度符合
    statement->row_to_insert.id = id;
    // 此处截取出来的字符串返回的事字符串指针，所以要通过strcpy将其完整的复制到结构体中
    strcpy(statement->row_to_insert.username,username);
    strcpy(statment->row_to_insert.email,email);
    
    return PREPARE_SUCCESS;
}


```

由于我们细化了一个字符串过长的错误，所以要向`PrepareResult`枚举中加入一个新的错误

``` {.c}
enum PrepareResult
{
    PREPARE_SUCCESS,
    PREPARE_SYNTAX_ERROR,
    PREPARE_UNRECONGNIZED_STATEMENT,
+   PREPARE_STRING_TOO_LONG, // 字符串过长
+   PREPARE_NEGATIVE_ID // ID不合法
};
```

接下来我们就可以在解析的过程中直接调用我们刚写的`prepare_insert()`函数了

``` {.c}
PrepareResult prepare_statement(InputBUffer* input_buffer,Statement* statement)
{
	if (strncmp(input_buffer->buffer,"insert",6) == 0)
    {
+       return prepare_insert(input_buffer,statement);
        // 原来if中的内容全部删除即可
    }
}
```

最后我们该如何告诉用户输入的数据不合法呢？通过向main()函数中swicth语句加入提示语句即可

``` {.c}
   Statement statement;
   switch (prepare_statement(input_buffer, &statement)) 
	{
      case (PREPARE_SUCCESS):
        break;
      case (PREPARE_SYNTAX_ERROR):
		printf("Syntax error. Could not parse statement.\n");
		continue;
      case (PREPARE_UNRECOGNIZED_STATEMENT):
        printf("Unrecognized keyword at start of '%s'.\n",input_buffer->buffer);
        continue;
           
+       case (PREPARE_NEGATIVE_ID):
+           printf("ID must be positive.\n");
+           continue;
+       case (PREPARE_STRING_TOO_LONG):
+           printf("String is too long.\n");
+           continue;
    }
```

## 完整差异

``` {.c}
 enum PrepareResult_t {
   PREPARE_SUCCESS,
+  PREPARE_NEGATIVE_ID,
+  PREPARE_STRING_TOO_LONG,
   PREPARE_SYNTAX_ERROR,
   PREPARE_UNRECOGNIZED_STATEMENT
  };
@@ -34,8 +36,8 @@
 #define COLUMN_EMAIL_SIZE 255
 typedef struct {
   uint32_t id;
-  char username[COLUMN_USERNAME_SIZE];
-  char email[COLUMN_EMAIL_SIZE];
+  char username[COLUMN_USERNAME_SIZE + 1];
+  char email[COLUMN_EMAIL_SIZE + 1];
 } Row;

@@ -150,18 +152,40 @@ MetaCommandResult do_meta_command(InputBuffer* input_buffer, Table *table) {
   }
 }

-PrepareResult prepare_statement(InputBuffer* input_buffer,
-                                Statement* statement) {
-  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+PrepareResult prepare_insert(InputBuffer* input_buffer, Statement* statement) {
   statement->type = STATEMENT_INSERT;
-  int args_assigned = sscanf(
-     input_buffer->buffer, "insert %d %s %s", &(statement->row_to_insert.id),
-     statement->row_to_insert.username, statement->row_to_insert.email
-     );
-  if (args_assigned < 3) {
+
+  char* keyword = strtok(input_buffer->buffer, " ");
+  char* id_string = strtok(NULL, " ");
+  char* username = strtok(NULL, " ");
+  char* email = strtok(NULL, " ");
+
+  if (id_string == NULL || username == NULL || email == NULL) {
      return PREPARE_SYNTAX_ERROR;
   }
+
+  int id = atoi(id_string);
+  if (id < 0) {
+     return PREPARE_NEGATIVE_ID;
+  }
+  if (strlen(username) > COLUMN_USERNAME_SIZE) {
+     return PREPARE_STRING_TOO_LONG;
+  }
+  if (strlen(email) > COLUMN_EMAIL_SIZE) {
+     return PREPARE_STRING_TOO_LONG;
+  }
+
+  statement->row_to_insert.id = id;
+  strcpy(statement->row_to_insert.username, username);
+  strcpy(statement->row_to_insert.email, email);
+
   return PREPARE_SUCCESS;
+
+}
+PrepareResult prepare_statement(InputBuffer* input_buffer,
+                                Statement* statement) {
+  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+      return prepare_insert(input_buffer, statement);
   }
   if (strcmp(input_buffer->buffer, "select") == 0) {
     statement->type = STATEMENT_SELECT;
@@ -223,6 +247,12 @@ int main(int argc, char* argv[]) {
     switch (prepare_statement(input_buffer, &statement)) {
       case (PREPARE_SUCCESS):
         break;
+      case (PREPARE_NEGATIVE_ID):
+	printf("ID must be positive.\n");
+	continue;
+      case (PREPARE_STRING_TOO_LONG):
+	printf("String is too long.\n");
+	continue;
       case (PREPARE_SYNTAX_ERROR):
 	printf("Syntax error. Could not parse statement.\n");
 	continue;
```

# 第五部分 - Persistence to Disk

该部分的任务目标是将我们之前写入内存的信息持久化进硬盘文件中，这不仅涉及到如何将内存中的数据全部写入文件，还涉及到下次启动数据库如何将本地文件中的数据加载进内存还有更重要的在于，若是一部分数据在内存（比如说刚刚新增的）一部分数据在本地文件中，此时该如何查找索引某条数据

回想最初说的SQLite数据库的结构，在后端（`back-end`）的几个重要构成部分中的一个**页缓冲（`Page Cache/Pager`）**的作用就是实现我们上述的需求，所以我们将在本部分完成`Pager`的定义与实现（具体配合`B-Tree`将在下部分完成）

我们将访问页面缓存和文件的总体功能交给Pager，所以我们要重构之前肩负着一部分该责任的结构体`Table`

## 初始化pager table

原`Table`结构体

``` {.c}
typedef struct
{
    void* pages[TABLE_MAX_PAGES]; // 存储着所有分配页的页指针数组
    uint32_t num_rows; // 目前存在的总数据条数
}Table;
```

为了方便`Pager`对页的管理与操作，我们页指针数组交予`Pager`管理

``` {.c}
typedef struct
{
    int file_descriptor; // 用于存放文件描述符
    uint32_t file_length; // 文件的大小
    void* pages[TABLE_MAX_PAGES];
}Pager;
```

并在Table结构体中包含该结构体

``` {.c}
typedef struct
{
	Pager* pager;
    uint32_t num_rows;
}Table;
```

这里我们定义了新的结构体，并且原Table结构体中的内容也发生了变化，所以接下来要解决的就是如何初始化这些数据

先来看一下之前我们初始化`Table`结构体的逻辑

``` {.c}
Table* new_table()
{
    Table* table = (Table*)malloc(sizeof(Table));
    table->num_rows = 0; // 初始时，页中数据的条数为0
    
    for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++)
    {
        table->pages[i] = NULL;
    }
    
    return table;
}
```

由于Table结构体中元素发生了变化，所以我们要更改上述代码，此时初始化Table结构体实际上要做的是：

-   打开数据库文件 `pager_open()`

-   初始化Pager `db_open()`

-   初始化Table `pager_open()`

``` {.c}
// 新的初始化函数由于兼顾了打开数据库文件的功能，所以我们将其重命名为db_open()

-	Table* new_table(){

+	Table* db_open(const char* filename)

	{

    	// 该函数将在接下来实现，用于打开数据库文件

		Pager* pager = pager_open(filename);

    	// 算出当前数据库文件已经存在总共多少条用户信息

    	uint32_t num_rows = pager->file_length / ROW_SIZE;

    

    	Table *table = malloc(sizeof(Table));

    	table->pager = pager;

    	table->num_rows = num_rows; 

    // 此时虽然没有将本地文件中的所有数据读回内存，但是已在内存中为其分配了“内存空间”即虽然没有将数据导	// 入，但这块空间已经被占用，并随时准备将所要查询的在本地文件中的数据放回内存对应的页中

    

    	return table;

	}

```

接下来我们来顺着实现用于打开文件的函数`pager_open()`

此处的I/O操作使用的均为一般用于低级 POSIX 文件描述符 I/O `open()`
`lseek()`

我们也可以使用更易于使用的stdio `fopen()` `fseek()`

``` {.c}
Pager* pager_open(const char* filename)
{
    int fd = open(filename,O_RDWR | O_CREAT,S_IWUSR | S_IRUSR);
    
    if (fd == -1) // 若打开失败
    {
        printf("Uable to open file.\n"); 
        // perror("open()");
        // 但此处用户无需知道过多细节，所以无需返回详细错误
        exit(EXIT_FAILURE);
    }
    
    // 获取打开文件的大小
    off_t file_length = lseek(fd,0,SEEK_END)
        
    Pager* pager = malloc(sizeof(Pager));
    pager->file_descriptor = fd;
    page->file_length = file_length;
    
    // 初始化页指针数组
    for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++)
    {
        pager->pager[i] = NULL;
    }
    
    return pager;
}
```

此处对于打开文件，我们也可以用`fopen()`实现，其参数会更简单
`fopen(filename,w)` 即可

![](https://raw.githubusercontent.com/De4tsh/typoraPhoto/main/img/202209160904019.png)

## 读取文件进内存

虽然我们在内存中数据是分页存储的，但最终存储到文件中时，是邻接存储的，也就是第0页的偏移量为0，那么第一页的偏移量
= 0 + 4096（页的大小），第二页的偏移自然也就是 4096 + 4096 =
8192以此类推

所以我们的逻辑是这样的，当我们查询某条数据时，若该数据并不在内存中时，我们的做法是将本地数据库文件加载进内存，但若我们要请求的页面位于文件边界之外，就说明他是空白的，此时我们只是分配一些内存并返回，当我们稍后退出数据库的时候，会将新增的数据的页刷新进本地文件中

``` {.c}
// 其中page_num代表着当前要查询的数据位于哪页中

// 接下来会重写row_slot，到时候就会清晰

void* get_page(Pager* pager,uint32_t page_num)

{

    // 若要找/添加的数据超过最大页限制

    if (page_num > TABLE_MAX_PAGES)

    {

        printf("Tired to fetch page number out of bounds.%d > %d\n",page_num,PAGE_MAX_PAGES);

        exit(EXIT_FAILURE);

    }

    

    // 若请求的页面不存在，在内存中分配空间，用于从本地数据库文件中加载

    // 因为数据库中的页对应关系和内存中是相同的

    if (pager->pages[page_num] == NULL)

    {	

        // 为其分配页空间

        void* page = malloc(PAGE_SIZE);

        // 当前本地文件中存在多少页的数据

        uint32_t num_pages = pager->file_length / PAGE_SIZE;

        

        if (pager->file_length % PAGE_SIZE)

        {

            // 因为我们要将本地数据库文件中的数据加载进内存，所以内存要为其分页以存			 				// 储这些数据，若数据库的文件整数页存放不下，则我们需要为其再多分配一页

            num_pages += 1;

		}

        

        // 若要读取的页在本地数据库文件中

        if (page_num <= num_pages)

        {

            // 将文件指针指向我们要读取页的起始处（从文件开头偏移要读取页的页号*页			 	 			 // 页的大小）

            lseek(pager->file_descriptor,page_num * PAGE_SIZE,SEEK_SET);

            

            // 将本页的数据读取到刚刚开辟的页空间page

            // 从pager->file_descriptor读取到*page

            ssize_t bytes_read = read(pager->file_descriptor,page,PAGE_SIZE);

            

            if (bytes_read == -1)

            {

                printf("Error reading file: %d\n",errno);

                exit(EXIT_FAILURE);

            }

            

        }

        

        pager->pages[page_num] = page;

	}

    

    return pager->pages[page_num];

}
```

现在我们可以将该功能合并进我们的`row_slot()`函数了（重构了原`row_slot()`函数）

``` {.c}
void* row_slot(Table* table,uint32_t row_num)
{
    uint32_t page_num = row_num / ROWS_PER_PAGE;
    
    void* page = get_page(table->pager,page_num);
    uint32_t row_offset = row_num % ROWS_PER_PAGE;
    uint32_t byte_offset = row_offset * ROW_SIZE;
    
    return page + byte_offset;
}
```

## 将缓存刷新进磁盘

当用户插入数据时，该数据将先保存在内存页中，在关闭数据库时这些缓存将被刷新进本地数据库文件中，所以我们专门定义一个函数`db_close()`用于实现：

-   将页面缓存刷新到本地数据库文件中

-   关闭数据库文件

-   释放Pager和Table

``` {.c}
void db_close(Table* table)
{

	Pager* page = table->pager;
    
    // 当前内存中有多少页
    uint32_t num_full_pages = table->num_rows / ROWS_PER_PAGE;
    
    for (uint32_t i = 0; i < num_full_pages; i++)
    {
        if (pager->pages[i] == NULL)
        {
            continue;
        }
        
        // 该函数用于将内存中的页写入本地文件中
        // 将在接下来实现该函数
        pager_flush(pager,i,PAGE_SIZE);
        
        free(pager->pages[i]);
        pager->pages[i] = NULL;
    }
    
    // 上面只是写入本地文件并释放了整数页的数据，若存在最后一页没写满的情况，需专门去释放
    // 该页
    uint32_t num_addition_rows = table->num_rows % ROWS_PER_PAGE;
    
    // 最后一页没有放满
    if (num_addtion_rows > 0)
    {
        uint32_t page_num = num_full_pages; // 最后一页对应在页指针数组中的编号
        if (pager->pages[page_num] != NULL)
        {
            pager_flush(pager,page_num,num_addtional_rows * ROW_SIZE);
            free(pager->pages[page_num]);
            pager->pages[page_num] = NULL;
        }
    }
    
    int result = close(pager->file_descriptor);
    if (result == -1)
    {
        printf("Error closing db file.\n");
        exit(EXIT_FAILURE);
    }
    
    for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++)
    {
        void* page = pager->pages[i];
        if (page)
        {
            free(page);
            pager->pages[i] = NULL;
        }
    }
    
    free(pager);
    free(table);
}
```

释放函数`pager_flush()`

当前的设计中，文件的长度编码了数据库中有多少行，所以我们需要在文件末尾写一个部分页面。这就是为什么`pager_flush()`需要一个页码和一个大小。这不是最好的设计，开始实现
B-tree 时它会很快消失。

``` {.c}
/*
page_num 内存页号
size 大小
*/
// 用于将内存的页写入本地数据库文件中
void pager_flush(Pager* pager,uint32_t page_num,uint32_t size)
{
	
    if (page->page[page_num] == NULL)
    {
        printf("Tired to flush null page.\n");
        exit(EXIT_FAILURE);
    }

    // 在这之前已经写入了page_num * PAGE_SIZE大小的数据了
    off_t offset = lseek(pager->file_descriptor,page_num * PAGE_SIZE,SEEK_SET);
    
    if (offset == -1)
    {
        printf("Error seeking: %d\n"errno);
        exit(EXIT_FAILURE);
    }
    
    ssize_t bytes_written = write(pager->descriptor,pager->pages[page_num],size);
    
    if (bytes_written == -1)
    {
        printf("Error writing: %d\n",errno);
        exit(EXIT_FAILURE);
    }

}
```

在输入`.exit`后调用`db_close()`即可

``` {.c}
-MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
+MetaCommandResult do_meta_command(InputBuffer* input_buffer, Table* table) {
   if (strcmp(input_buffer->buffer, ".exit") == 0) {
+    db_close(table);
     exit(EXIT_SUCCESS);
   } else {
     return META_COMMAND_UNRECOGNIZED_COMMAND;
```

## main()函数 {#main函数-2}

最后，我们需要接受文件名作为命令行参数。不要忘记还将额外的参数添加到`do_meta_command`

``` {.c}
 int main(int argc, char* argv[]) {
-  Table* table = new_table();
+  if (argc < 2) {
+    printf("Must supply a database filename.\n");
+    exit(EXIT_FAILURE);
+  }
+
+  char* filename = argv[1];
+  Table* table = db_open(filename);
+
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);

     if (input_buffer->buffer[0] == '.') {
-      switch (do_meta_command(input_buffer)) {
+      switch (do_meta_command(input_buffer, table)) {
```

## 完全差异 {#完全差异-1}

``` {.c}
+#include <errno.h>
+#include <fcntl.h>
 #include <stdbool.h>
 #include <stdio.h>
 #include <stdlib.h>
 #include <string.h>
 #include <stdint.h>
+#include <unistd.h>

 struct InputBuffer_t {
   char* buffer;
@@ -62,9 +65,16 @@ const uint32_t PAGE_SIZE = 4096;
 const uint32_t ROWS_PER_PAGE = PAGE_SIZE / ROW_SIZE;
 const uint32_t TABLE_MAX_ROWS = ROWS_PER_PAGE * TABLE_MAX_PAGES;

+typedef struct {
+  int file_descriptor;
+  uint32_t file_length;
+  void* pages[TABLE_MAX_PAGES];
+} Pager;
+
 typedef struct {
   uint32_t num_rows;
-  void* pages[TABLE_MAX_PAGES];
+  Pager* pager;
 } Table;

@@ -84,32 +94,81 @@ void deserialize_row(void *source, Row* destination) {
   memcpy(&(destination->email), source + EMAIL_OFFSET, EMAIL_SIZE);
 }

+void* get_page(Pager* pager, uint32_t page_num) {
+  if (page_num > TABLE_MAX_PAGES) {
+     printf("Tried to fetch page number out of bounds. %d > %d\n", page_num,
+     	TABLE_MAX_PAGES);
+     exit(EXIT_FAILURE);
+  }
+
+  if (pager->pages[page_num] == NULL) {
+     // Cache miss. Allocate memory and load from file.
+     void* page = malloc(PAGE_SIZE);
+     uint32_t num_pages = pager->file_length / PAGE_SIZE;
+
+     // We might save a partial page at the end of the file
+     if (pager->file_length % PAGE_SIZE) {
+         num_pages += 1;
+     }
+
+     if (page_num <= num_pages) {
+         lseek(pager->file_descriptor, page_num * PAGE_SIZE, SEEK_SET);
+         ssize_t bytes_read = read(pager->file_descriptor, page, PAGE_SIZE);
+         if (bytes_read == -1) {
+     	printf("Error reading file: %d\n", errno);
+     	exit(EXIT_FAILURE);
+         }
+     }
+
+     pager->pages[page_num] = page;
+  }
+
+  return pager->pages[page_num];
+}
+
 void* row_slot(Table* table, uint32_t row_num) {
   uint32_t page_num = row_num / ROWS_PER_PAGE;
-  void *page = table->pages[page_num];
-  if (page == NULL) {
-     // Allocate memory only when we try to access page
-     page = table->pages[page_num] = malloc(PAGE_SIZE);
-  }
+  void *page = get_page(table->pager, page_num);
   uint32_t row_offset = row_num % ROWS_PER_PAGE;
   uint32_t byte_offset = row_offset * ROW_SIZE;
   return page + byte_offset;
 }

-Table* new_table() {
-  Table* table = malloc(sizeof(Table));
-  table->num_rows = 0;
+Pager* pager_open(const char* filename) {
+  int fd = open(filename,
+     	  O_RDWR | 	// Read/Write mode
+     	      O_CREAT,	// Create file if it does not exist
+     	  S_IWUSR |	// User write permission
+     	      S_IRUSR	// User read permission
+     	  );
+
+  if (fd == -1) {
+     printf("Unable to open file\n");
+     exit(EXIT_FAILURE);
+  }
+
+  off_t file_length = lseek(fd, 0, SEEK_END);
+
+  Pager* pager = malloc(sizeof(Pager));
+  pager->file_descriptor = fd;
+  pager->file_length = file_length;
+
   for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++) {
-     table->pages[i] = NULL;
+     pager->pages[i] = NULL;
   }
-  return table;
+
+  return pager;
 }

-void free_table(Table* table) {
-  for (int i = 0; table->pages[i]; i++) {
-     free(table->pages[i]);
-  }
-  free(table);
+Table* db_open(const char* filename) {
+  Pager* pager = pager_open(filename);
+  uint32_t num_rows = pager->file_length / ROW_SIZE;
+
+  Table* table = malloc(sizeof(Table));
+  table->pager = pager;
+  table->num_rows = num_rows;
+
+  return table;
 }

 InputBuffer* new_input_buffer() {
@@ -142,10 +201,76 @@ void close_input_buffer(InputBuffer* input_buffer) {
   free(input_buffer);
 }

+void pager_flush(Pager* pager, uint32_t page_num, uint32_t size) {
+  if (pager->pages[page_num] == NULL) {
+     printf("Tried to flush null page\n");
+     exit(EXIT_FAILURE);
+  }
+
+  off_t offset = lseek(pager->file_descriptor, page_num * PAGE_SIZE,
+     		 SEEK_SET);
+
+  if (offset == -1) {
+     printf("Error seeking: %d\n", errno);
+     exit(EXIT_FAILURE);
+  }
+
+  ssize_t bytes_written = write(
+     pager->file_descriptor, pager->pages[page_num], size
+     );
+
+  if (bytes_written == -1) {
+     printf("Error writing: %d\n", errno);
+     exit(EXIT_FAILURE);
+  }
+}
+
+void db_close(Table* table) {
+  Pager* pager = table->pager;
+  uint32_t num_full_pages = table->num_rows / ROWS_PER_PAGE;
+
+  for (uint32_t i = 0; i < num_full_pages; i++) {
+     if (pager->pages[i] == NULL) {
+         continue;
+     }
+     pager_flush(pager, i, PAGE_SIZE);
+     free(pager->pages[i]);
+     pager->pages[i] = NULL;
+  }
+
+  // There may be a partial page to write to the end of the file
+  // This should not be needed after we switch to a B-tree
+  uint32_t num_additional_rows = table->num_rows % ROWS_PER_PAGE;
+  if (num_additional_rows > 0) {
+     uint32_t page_num = num_full_pages;
+     if (pager->pages[page_num] != NULL) {
+         pager_flush(pager, page_num, num_additional_rows * ROW_SIZE);
+         free(pager->pages[page_num]);
+         pager->pages[page_num] = NULL;
+     }
+  }
+
+  int result = close(pager->file_descriptor);
+  if (result == -1) {
+     printf("Error closing db file.\n");
+     exit(EXIT_FAILURE);
+  }
+  for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++) {
+     void* page = pager->pages[i];
+     if (page) {
+         free(page);
+         pager->pages[i] = NULL;
+     }
+  }
+
+  free(pager);
+  free(table);
+}
+
 MetaCommandResult do_meta_command(InputBuffer* input_buffer, Table *table) {
   if (strcmp(input_buffer->buffer, ".exit") == 0) {
     close_input_buffer(input_buffer);
-    free_table(table);
+    db_close(table);
     exit(EXIT_SUCCESS);
   } else {
     return META_COMMAND_UNRECOGNIZED_COMMAND;
@@ -182,6 +308,7 @@ PrepareResult prepare_insert(InputBuffer* input_buffer, Statement* statement) {
     return PREPARE_SUCCESS;

 }
+
 PrepareResult prepare_statement(InputBuffer* input_buffer,
                                 Statement* statement) {
   if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
@@ -227,7 +354,14 @@ ExecuteResult execute_statement(Statement* statement, Table *table) {
 }

 int main(int argc, char* argv[]) {
-  Table* table = new_table();
+  if (argc < 2) {
+      printf("Must supply a database filename.\n");
+      exit(EXIT_FAILURE);
+  }
+
+  char* filename = argv[1];
+  Table* table = db_open(filename);
+
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
```

# 第六部分：The Cursor Abstraction

## Cursor概念

为了方便我们后续更轻松地实现B-Tree，此处我们将实现游标的功能，在实现以前，先从WIKI中复制一段关于数据库游标的描述，方便后续理解其含义

![](https://raw.githubusercontent.com/De4tsh/typoraPhoto/main/img/202209171002626.png)

游标是一种能够遍历数据库中记录的机制，游标有助于与遍历一起进行后续操作，例如数据库记录的检索、添加和删除

游标可以看作是只想一组行中的一行的指针

所以，我们将添加一个`游标(Cursor)`表示表中位置的对象，具备以下功能：

-   在表的开头创建一个游标

-   在表的末尾创建一个游标

-   访问游标所指向的行

-   移动游标到下一行（Row）

另外之后我们还要用其完成的操作有：

-   删除游标指向的行（Row）

-   修改游标所指向的行（Row）

-   在表中搜索给定的ID，并创建一个指向该ID的游标

## Cursor结构

``` {.c}
typedef struct

{

    Table* table;

    uint32_t row_num;

    bool end_of_table;   

}Cursor;
```

-   `row_num` 用于标识当前要操作位置的行号

-   `end_of_table`
    当前位置是否为已存在数据行的末尾（一般插入新的数据插入在末尾）

-   `table` 所属表的引用

## 创建新游标

之前需具备的功能里提过，我们需要创建一个**指向表开头的游标**和一个**指向表结束的游标**，所以我们定义两个函数`table_start()`
和 `table_end()`用于创建这两个游标

``` {.c}
Cursor* table_start(Table* table)

{

    Cursor* cursor = malloc(sizeof(Cursor));

    cursor->table = table;

    cursor->row_num = 0;

    cursor->end_of_table = (table->nums_rows == 0); 

    // 若当前总数据行数为0，就表示当前既是开头也是末尾

    

    return cursor;

}
```

``` {.c}
Cursor* table_end(Table* table)

{

    Cursor* cursor = malloc(sizeof(Cursor));

    cursor->table = table;

    cursor->row_num = table->num_rows;

    cursor->end_of_table = true;

    

    return cursor;

}
```

## 通过游标索引对应行

之前我们索引对应行的函数是`void* row_slot(Table* table,uint32_t row_num)`,我们先将原来的函数拿来看一下（具体注释在上一部分中查看即可）

``` {.c}
void* row_slot(Table* table,uint32_t row_num)

{

	uint32_t page_num = row_num / ROWS_PER_PAGE;

    

    // 确认了页号后，根据在页指针数组中找到该页对应的指针

    void* page = table -> pages[page_num]

        

   	if (page == NULL)

    {

        page = table->pages[page_num] = malloc(PAGE_SIZE)

    }



    uint32_t row_offset = row_num % ROWS_PER_PAGE;

    

    // 当前页已经存在的数据条数 * 每条数据的大小 = 要存新数据的起始字节偏移量

    uint32_t byte_offset = row_offset * ROW_SIZE;

    

    return page + byte_offset;

}
```

我们可以看到该函数的参数中无论是表的引用`Table* table`还是要定位的数据行`num_row`
都在我们的新结构游标中进行了聚合，所以我们改写该函数，只要传入游标即可

我们重命名`row_slot()`为`cursor_value()`因为该函数用于返回游标指向的位置

``` {.c}
void* cursor_value(Cursor* cursor)

{

    uint32_t row_num = cursor->row_num;

    uint32_t page_num = row_num / ROWS_PER_PAGE;

    

    void* page = get_page(cursor->table->pager,page_num);

    

    uint32_t row_offset = row_num % PAGE_PER_PAGE;

    uint32_t bytes_offset = row_offset * ROW_SIZE;

    

    return page + byte_offset;

}
```

接下来如果我们想索引该数据对应的下一条数据只需将`Cursor`中的`row_num+1`即可，那么再下次查询的时候就会查询下一条数据

``` {.c}
void cursor_advance(Cursor* cursor)

{

    cursor->row_num += 1;

    

    // 若下一条数据已经超过了当前存在的数据的条数，就说明当前已经到表尾了

    if (cursor->row_num >= cursor->table->num_rows)

    {

        cursor->end_of_table = true;

    }

}
```

有了该操作，我们就有了一个大胆的想法，那么之前的无论是序列化函数还是反序列化函数，是不是都可以通过游标来不断往下一条数据切换了，所以我们重构上述代码的参数

为了防止我们写的写的晕了，首先把原始的虚拟机执行`insert`的代码拿过来

``` {.c}


+ExecuteResult execute_insert(Statement* statement,Table* table)

{

    // 若当前页表中所有的页都已满，则不能再插入了

    if (table->num_rows >= TABLE_MAX_ROWS)

    {

        return EXECUTE_TABLE_FULL;

    }

    

    // 提示：statement->row_to_insert中存放着用户要插入的详细数据

    Row* row_to_insert = &(statement->row_to_insert);

    

    // 将序列化的后的数据写入页中

    serialize_row(row_to_insert,row_slot(table,table->num_rows));

    

    table->nums_rows += 1; // 插入了一条数据后，总的数据条数+1

    

    return EXECUTE_SUCCESS;

}
```

重构该代码，加入游标

``` {.c}
ExecuteResult execute_insert(Statement* statement,Table* table)

{

 

    if (table->num_rows >= TABLE_MAX_ROWS)

    {

        return EXECUTE_TABLE_FULL;

    }

    



    Row* row_to_insert = &(statement->row_to_insert);

    

    Cursor* cursor = table_end(table); // 由于要插入数据，所以指向表尾部

    



    serialize_row(row_to_insert,cursor_value(cursor));  // 向表的尾部插入新数据

    

    table->nums_rows += 1; 

    

    free(cursor); 

    // 使用完释放游标即可

    // 由于游标是从table->nums_row中获取总数据数，所以不用担心其中的数据消失

    // 只要table没有被释放，游标每次创建都可以动态获取最新的位置关系

    

    return EXECUTE_SUCCESS;

}
```

当然，对于`select`我们也需要进行修改，老规矩，先把以前的`select`的实现代码拿来

``` {.c}
ExecuteResult execute_select(Statement* statement,Table* table)

{

    Row row;

    for (uint32_t i = 0; i < table->num_rows; i++)

    {

        // 将内存中序列化后的数据反序列化为结构体中的数据结构

        deserialize_row(row_slot(table,i),&row);

        print_row(&row);

    }

    

    return EXECUTE_SUCCESS;

}
```

重构该代码，加入游标

``` {.c}
ExecuteResult execute_select(Statement* statement,Table* table)

{

    Cursor* cursor = table_start(table); // 初始化游标只想表头

    

    Row row;

    // 无需在进行for循环，使用我们之前用于移动游标的cursor_advance()函数即可

    // for (uint32_t i = 0; i < table->num_rows; i++)

    // {

    

    // 只要没有到表尾

    while(!(cursor->end_of_table))

    {

        deserialize_row(cursor_value(cursor),&row);

        print_row(&row);

        cursor_advance(cursor); // 移动游标指向下一条数据

    }

        

        free(cursor);

        

    // }

    

    return EXECUTE_SUCCESS;

}
```

## 完全差异 {#完全差异-2}

``` {.c}
@@ -78,6 +78,13 @@ struct {

 } Table;



+typedef struct {

+  Table* table;

+  uint32_t row_num;

+  bool end_of_table; // Indicates a position one past the last element

+} Cursor;

+

 void print_row(Row* row) {

     printf("(%d, %s, %s)\n", row->id, row->username, row->email);

 }

@@ -126,12 +133,38 @@ void* get_page(Pager* pager, uint32_t page_num) {

     return pager->pages[page_num];

 }



-void* row_slot(Table* table, uint32_t row_num) {

-  uint32_t page_num = row_num / ROWS_PER_PAGE;

-  void *page = get_page(table->pager, page_num);

-  uint32_t row_offset = row_num % ROWS_PER_PAGE;

-  uint32_t byte_offset = row_offset * ROW_SIZE;

-  return page + byte_offset;

+Cursor* table_start(Table* table) {

+  Cursor* cursor = malloc(sizeof(Cursor));

+  cursor->table = table;

+  cursor->row_num = 0;

+  cursor->end_of_table = (table->num_rows == 0);

+

+  return cursor;

+}

+

+Cursor* table_end(Table* table) {

+  Cursor* cursor = malloc(sizeof(Cursor));

+  cursor->table = table;

+  cursor->row_num = table->num_rows;

+  cursor->end_of_table = true;

+

+  return cursor;

+}

+

+void* cursor_value(Cursor* cursor) {

+  uint32_t row_num = cursor->row_num;

+  uint32_t page_num = row_num / ROWS_PER_PAGE;

+  void *page = get_page(cursor->table->pager, page_num);

+  uint32_t row_offset = row_num % ROWS_PER_PAGE;

+  uint32_t byte_offset = row_offset * ROW_SIZE;

+  return page + byte_offset;

+}

+

+void cursor_advance(Cursor* cursor) {

+  cursor->row_num += 1;

+  if (cursor->row_num >= cursor->table->num_rows) {

+    cursor->end_of_table = true;

+  }

 }



 Pager* pager_open(const char* filename) {

@@ -327,19 +360,28 @@ ExecuteResult execute_insert(Statement* statement, Table* table) {

     }



   Row* row_to_insert = &(statement->row_to_insert);

+  Cursor* cursor = table_end(table);



-  serialize_row(row_to_insert, row_slot(table, table->num_rows));

+  serialize_row(row_to_insert, cursor_value(cursor));

   table->num_rows += 1;



+  free(cursor);

+

   return EXECUTE_SUCCESS;

 }



 ExecuteResult execute_select(Statement* statement, Table* table) {

+  Cursor* cursor = table_start(table);

+

   Row row;

-  for (uint32_t i = 0; i < table->num_rows; i++) {

-     deserialize_row(row_slot(table, i), &row);

+  while (!(cursor->end_of_table)) {

+     deserialize_row(cursor_value(cursor), &row);

      print_row(&row);

+     cursor_advance(cursor);

   }

+

+  free(cursor);

+

   return EXECUTE_SUCCESS;

 }
```
B-Tree是SQLite用来表示表和索引的数据结构，也是接下来及部分我们核心要完成的数据结构

![B-tree](https://raw.githubusercontent.com/De4tsh/typoraPhoto/main/img/202209191458364.png)

（https://en.wikipedia.org/wiki/File:B-tree.svg）`B-Tree`

上面为B树的结构图，具体B树的优点、效率我们就不在此处讨论了，我们重在写出其实际的实现

我们接下来要实现的是B树的一种变体**B+树**，这也是SQLite使用的数据结构，这里简单引用原文对B+ Tree的图例，方便理解

![btree6](https://raw.githubusercontent.com/De4tsh/typoraPhoto/main/img/202209191505132.png)

图示为一张3级B+ Tree，其中蓝色部分为内部结点不存数据只存索引与叶子节点的指针，绿色部分则位叶子结点，存放着键值对（我们的数据）

## 页结点格式

### 之前存储结构的不足

若我们不使用树结构，则当前格式每个页面仅存放数据行，没有任何的额外数据（元数据），非常节约空间，向末尾追加新的数据行也会非常快。

但是一旦遇到查找，效率就会降下来，因为我们需要扫描整个表来完成；删除同样会遇到问题，由于我们是顺序存储，所以删除其中的某一条数据势必要将其后的所有数据前移，影响效率；在数据中间插入新数据同样会面临这个问题

因此我们**将使用树结构来重构我们的数据存储格式，树中每个节点都可以包含可变数量的行，所以我们不得不在每个节点页中存储一些信息来跟踪他的状态**

这也是一种取舍，我们通过空间换取了时间，**舍弃了为节点页增加描述信息，以及内部节点（不存储数据）对空间的占用这些额外空间，换来了更高效的插入、删除和查找**。

### 节点头格式

> 节点类型分为 **叶子节点** 和 **内部节点**，我们通过建立一个枚举类型来跟踪节点类型

```C
typedef enum
{
    NODE_INTERNAL, // 内部节点
    NODE_LEAF // 叶子节点
}NodeType;
```

**每个节点对应一页**，**内部节点将通过存储子节点的页码来指向它们的子节点**。B+ Tree向Pager请求特定的页号，并取回一个指向页缓存的指针。页面按页码顺序一个接一个地存储在数据库文件中

为了防止忘记Pager的作用，我们再将其结构体拿来看一下：

```C
typedef struct {
  int file_descriptor; // 文件描述符
  uint32_t file_length; // 文件大小长度
  void* pages[TABLE_MAX_PAGES]; // 页指针数组
} Pager;

```

为完成上述的功能，我们需要向每个页（也就是一个节点）的开头的标头处存储一些元数据：

- 该节点是什么类型的节点
- 该节点是否为根节点
- 指向父亲节点的指针

#### 通用节点标头布局

> 无论是叶子节点还是内部节点都应在页的开始包含该标头

```C
// 节点类型
const uint32_t NODE_TYPE_SIZE = sizeof(uint8_t); // 一个字节
const uint32_t NODE_TYPE_OFFSET = 0; // 处于页的最开头第一个字节

// 是否为根节点
const uint32_t IS_ROOT_SIZE = sizeof(uint8_t); // 一个字节
const uint32_t IS_ROOT_OFFSET = NODE_TYPE_SIZE; // 紧跟节点类型的下一个字节

// 指向父亲节点的指针
const uint32_t PARENT_POINTER_SIZE = sizeof(uint32_t); // 4字节 32位指针
const uint32_t PARENT_PIONTER_OFFSET = IS_ROOT_OFFSET + IS_ROOT_SIZE; // 紧跟上一字节

// 通用节点标头所占的总空间
const uint8_t COMMON_NODE_HEADER_SIZE = NODE_TYPE_SIZE + IS_ROOT_SIZE + PARENT_POINTER_SIZE; // 上述三个基本信息一共占 1 + 1 + 4 = 6字节

```

#### 叶节点标头格式布局

> 叶节点的头部表示信息紧跟着通用节点标头之后

叶节点用于存储数据，叶节点的主体是单元格，由许多单元格组成，每一个单元格都是一个键/值（`key/value`）对，其中值就是我们之前序列化后的`Row`

所以在头部我们需要表示该叶节点包含多少单元

```C
// 最多容纳的单元格数量为4字节大小
const uint32_t LEAF_NODE_NUM_CELLS_SIZE = sizeof(uint32_t); 
// 紧跟在通用节点标头之后
const uint32_t LEAF_NODE_NUM_CELLS_OFFSET = COMMON_NODE_HEADER_SIZE; 

// 叶节点标头 + 通用节点标头所占据的总大小
const uint32_t LEAF_NODE_HEADER_SIZE = COMMON_NODE_HEADER_SIZE + LEAF_NODE_NUM_CELLS_SIZE;
```

#### 叶节点主体格式布局

如刚刚所说，叶节点的主题由单元格构成，所以我们接下来要来规定单元格的结构

```C
// 键所占的空间及偏移 key
const uint32_t LEAF_NODE_KEY_SIZE = sizeof(uint32_t);
const uint32_t LEAF_NODE_KEY_OFFSET = 0;

// 值所占的空间及其偏移 value
const uint32_t LEAF_NODE_VALUE_SIZE = ROW_SIZE; // 之前定义过，表示一条用户数据的大小
const uint32_t LEAF_NODE_VALUE_OFFSET = LEAF_NODE_KEY_OFFSET + LEAF_NODE_KEY_SIZE;

// 一个单元格的大小 key + vlaue
const uint32_t LEAF_NODE_CELL_SIZE = LEAF_NODE_KEY_SIZE + LEAF_NODE_VALUE_SIZE;

// 在结尾处会出现容纳不下一个完整单元格的冗余空间
// 一个页中出去通用表头和页标头后剩余的空间
const uint32_t LEAF_NODE_SPACE_FOR_CELLS = PAGE_SIZE - LEAF_NODE_HEADER_SIZE;

//一个页除去上述所说的标头信息外能存储的最多单元格个数
const uint32_t LEAF_NODE_MAX_CELLS = LEAF_NODE_SPACE_FOR_CELLS / LEAF_NODE_CELL_SIZE;
```

### 最终的总体节点页结构

![WeChat Screenshot_20220919183946](https://raw.githubusercontent.com/De4tsh/typoraPhoto/main/img/202209191841965.png)

## 访问叶节点的属性信息

既然我们刚刚在每个节点（页）的开头定义了一些描述信息，那么我们页需要提供额外的接口用于获取这些描述信息

### 访问叶节点字段

```C
// 获取node指向的叶节点标头的信息
// 用于获取该叶节点中单元格的个数
uint32_t leaf_node_num_cells(void* node)
// node指向一个页（节点）的开始
{
    return node + LEAF_NODE_NUM_CELLS_OFFSET;
}

// 定位到页节点中第cell_num单元格，返回其指针
void* leaf_node_cell(void* node,uint32_t cell_num)
{
    return node + LEAF_NODE_HEADER_SIZE + cell_num * LEAF_NODE_CELL_SIZE;
}

// 该函数意在获取单元格数据键值对中key的地址
// 但由于key值的地址其实就是一个单元格的起始地址，所以此函数站在功能的  // 角度上并没有存在的必要，但站在封装的角度上还是有必要的 
uint32_t* leaf_node_key(void* node,uint32_t cell_num)
{
    return leaf_node_cell(node,cell_num);
}

// 获取对应单元格value的地址
uint32_t leaf_node_value(void* node,uint32_t cell_num)
{
    return leaf_node_cell(node,cell_num) + LEAF_NODE_KEY_SIZE;
}

// 初始化单元格个数 = 0
void initialize_leaf_node(void* node)
{
    
    // 相当于 (node + LEAF_NODE_NUM_CELLS_OFFSET) = 0
    *leaf_node_num_cells(node) = 0;
}
```

### Pager/Table对象结构与操作方法的修改

先来回顾以下 `Pager` 和 `Table` 的结构体：

```C
typedef struct
{
    int file_descriptor; // 用于存放文件描述符
    uint32_t file_length; // 文件的大小
    void* pages[TABLE_MAX_PAGES];
}Pager;
```

```C
typedef struct
{
	Pager* pager;
    uint32_t num_rows;
}Table;
```

之前我们再数据库关闭时（db_close()）要将内存中的数据刷入本地数据库文件中，使用的方法是 `pager_flush()` 为防止遗忘，我们拿来看一下

```C
void pager_flush(Pager* pager,uint32_t page_num,uint32_t size)
{
	
    if (page->page[page_num] == NULL)
    {
        printf("Tired to flush null page.\n");
        exit(EXIT_FAILURE);
    }

    // 在这之前已经写入了page_num * PAGE_SIZE大小的数据了
    off_t offset = lseek(pager->file_descriptor,page_num * PAGE_SIZE,SEEK_SET);
    
    if (offset == -1)
    {
        printf("Error seeking: %d\n"errno);
        exit(EXIT_FAILURE);
    }
    
    ssize_t bytes_written = write(pager->descriptor,pager->pages[page_num],size);
    
    if (bytes_written == -1)
    {
        printf("Error writing: %d\n",errno);
        exit(EXIT_FAILURE);
    }

}
```

当时说过，该函数的设计是存在缺陷的，因为他要规定写入数据的`size`，这非常不符合我们按页进行存取的理念，**当时这么做是因为最后一页中会有零散的数据（所谓的零散就是没有填满一个页）**，所以我们需要通过`size`值传递零散数据的条数，达到将其写入本地数据库文件的目的

而现在由于我们引入了B+ Tree，所以一切都以节点为单位，一个节点正好是一页，此时我们便无需指定`size`，因为`size`固定都是`PAGE_SIZE`，即使没有填满我们也将写入一个页

新版`pager_flush()`

```C
void pager_flush(Pager* pager,uint32_t page_num)
{
    if (pager->pages[page_num] == NULL)
    {
        printf("Tired to flush null page\n");
        exit(EXIT_FAILURE);
    }
    
    ssize_t bytes_written = write(pager->descriptor,pager->pages[page_num],PAGE_SIZE);
    
    if (bytes_written == -1)
    {
        printf("Error writing:%d\n",errno);
    }
}
```

显得非常的清爽，同样我们也需要修改调用`pager_flush()`的函数db_close()

```C
void db_close(Table* table)
{
    Pager* pager = table->pager;
    
    // 将内存中存在页（节点）的内容写入本地数据库文件
    for (uint32_t i = 0;i < pager->num_pages; i++)
    {
        if (pager->pages[i] == NULL)
        {
            continue;
		}
        
        pager_flush(pager,i);
        free(pager->page[i]);
        pager->pages[i] = NULL
    }
    
    int result = close(pager->file_descriptor);
    if (result == -1)
    {
        printf("Error closing db file.\n");
    }
}
```

相比于以前那个版本，还需要判断最后有没有没填满一个页的数据，再单独写入的版本，此时按节点操作后，将变得无比的简单

现在由于我们在真正的按页操作（节点），每个页的标头信息中也会存储当前页数据量等信息，因此现在，**我们存储一个页面有多少行、以及所有页能存储多少条数据变得没有意义**

```C
// 删除如下已经没有意义的定义
-	const uint32_t ROWS_PER_PAGE = PAGE_SIZE / ROW_SIZE;
-	const uint32_t TABLE_MAX_ROWS = ROWS_PER_PAGE * TABLE_MAX_PAGES;
```

取而代之的我们应当在结构体中定义出当前存在的节点数（页数）

```C
typedef struct
{
    int file_descriptor;
    uint32_t file_length;
    uint32_t num_pages; // 新增节点个数
    void* pages[TABLE_MAX_PAGES];
}

typedef struct
{
    Pager* pager;
    uint32_t root_page_num; // ！！！新增根节点的页号！！！
}Table;
```

麻烦的事情来了，由于我们修改了该结构体，所以使用过上述删除参数的函数都要重写，并为其安排上新的参数信息，不着急我们一个一个修改

首先第一个是 `void* get_page(Pager* pager, uint32_t page_num)`

```C
        }
        
        pager->pages[page_num] = page;

		// 此时由于节点数发生了增加，所以我们需要更新num_pages值
		if (page_num >= pager->num_pages)
        {
            pager->num_pages = page_num +1;
        }
	}
    
    return pager->pages[page_num];
}
```

首先第二个是 `Pager* pager_open(const char* filename)` 其原函数为：

```C
Pager* pager_open(const char* filename)
{
    int fd = open(filename,O_RDWR | O_CREAT,S_IWUSR | S_IRUSR);
    
    if (fd == -1) // 若打开失败
    {
        printf("Uable to open file.\n"); 
        // perror("open()");
        // 但此处用户无需知道过多细节，所以无需返回详细错误
        exit(EXIT_FAILURE);
    }
    
    // 获取打开文件的大小
    off_t file_length = lseek(fd,0,SEEK_END)
        
    Pager* pager = malloc(sizeof(Pager));
    pager->file_descriptor = fd;
    page->file_length = file_length;
    
+	pager->num_pages = (file_length / PAGE_SIZE);
+	
+	if (file_length % PAGE_SIZE != 0)
+	{
+    	// 由于我们存储的时候都是按页存储的，所以若是读取时
+   	// 有一页未满的数据，就说明数据库文件发生了错误
+    	printf("Db file is not a whole number of pages.\n");
+    	exit(EXIT_FAILURE);
+	}
+
+
    
    
    
    // 初始化页指针数组
    for (uint32_t i = 0; i < TABLE_MAX_PAGES; i++)
    {
        pager->pager[i] = NULL;
    }
    
    return pager;
}
```


