# Linux命令行解析函数getopt_long()

Linux环境下大多数程序被编译成带参数运行的可执行程序，C语言提供了几个命令行参数解析函数来辅助完成工作。

## getopt_long()

getopt()只能处理短选项，而getopt_long()由于长选项结构体`option`返回的原因可以同时处理长选项和短选项。

```c
#include <unistd.h>
extern char *optarg;
extern int optind, opterr, optopt;

#include <getopt.h>
int getopt(int argc, char * const argv[],const char *optstring);
int getopt_long(int argc, char * const argv[], const char *optstring, const struct option *longopts, int *longindex);
int getopt_long_only(int argc, char * const argv[], const char *optstring, const struct option *longopts, int *longindex);
```

- `argc`：参数个数

- `argv`：参数内容

- `optstring`：表示短选项字符串。

  形式如`a:b::cd:`，分别表示程序支持的命令行短选项有-a、-b、-c、-d，冒号含义如下：

  - 只有一个字符，不带冒号——只表示选项， 如：`-c `
  - 一个字符，后接一个冒号——表示选项后面带一个参数，如：`-a 100`
  - 一个字符，后接两个冒号——表示选项后面带一个可选参数，即参数可有可无， 如果带参数，则选项与参数直接不能有空格，如：`-b10`

- `longopts`：长选项结构体

  ```c
  struct option 
  {  
       const char *name;  
       int         has_arg;  
       int        *flag;  
       int         val;  
  };
  ```

  (1)`name`：表示选项的名称,比如daemon,dir,out等
  
  (2)`has_arg`：表示选项后面是否携带参数，该参数有三个不同值，如下：
  
  - no_argument(或者是0)时   ——参数后面不跟参数值
  
  - required_argument(或者是1)时 ——参数输入格式为：--参数 值 或者 --参数=值
  
  - optional_argument(或者是2)时  ——参数输入格式只能为：--参数=值
  
  (3)`flag`：
  
  - 如果参数为空NULL，那么当选中某个长选项的时候，getopt_long将返回val值（这种情况下val通常被定义为短选项值）
  
  - 如果参数不为空，那么当选中某个长选项的时候，getopt_long将返回0，并且将flag指针参数指向val值。
  
  (4)`val`：表示指定函数找到该选项时的返回值，或者当flag非空时指定flag指向的数据的值val
  
- `longindex`：当找到对应选项时返回的是该选项在longopts结构体数组中的下标

- 全局变量：

  - `optarg`：当前选项对应的参数值
  - `optind`：下一个将被处理到的参数在argv中的下标值
  - `opterr`：getopt()，getopt_long()函数错误信息
  - `optopt`：最后一个未知选项

- 返回值：

  - 短选项找到，返回短选项对应的字符
  - 长选项找到，flag为NULL则返回val，flag不为NULL则返回0
  - 未找到返回-1

```c
#include <stdio.h>
#include <unistd.h>
#include <getopt.h>
int main()
{
	while(1) {
        int c;
        int lopt;
        struct option long_opts[] {
            {"input_img", required_argument, NULL, 'i'},
            {"input_fmt", required_argument, NULL, 's'},
            {"output_img", required_argument, NULL, 'o'},
            {"output_fmt", required_argument, NULL, 'f'},
            {"width", required_argument, NULL, 'w'},
            {"height", required_argument, NULL, 'd'},
            {"test", required_argument, &lopt, 1},
            {"help", no_argument, NULL, 'h'},
            {NULL , 0, NULL, 0}		// option结构体数组结束标志，不可省略
    	};
        c = getopt_long(argc, argv, "i:s:o:f:w:d:h", long_opts, &option_index);

        if (c == -1) break;

        switch (c)
        {
        case 0:
            switch(lopt)
            case 1:
            	// 
                break;
        case 'i':
            image.path = optarg;
            break;
        case 's':
            image.src_fmt = (pix_fmt)atoi(optarg);
            break;
        case 'o':
            output_path = optarg;
            break;
        case 'f':
            image.dst_fmt = (pix_fmt)atoi(optarg);
            break;
        case 'w':
            image.width = atoi(optarg);
            break;
        case 'd':
            image.height = atoi(optarg);
            break;
        case 'h':
        default:
            usage();
            exit(-1);
        }
    }
    return 0;
}
```

