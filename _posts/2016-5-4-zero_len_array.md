##code
#include <stdio.h>
#include <stdlib.h>

typedef struct STR_INT_A {
	int len;
	char buf[0];
}INT_A;

typedef struct STR_INT_B
{
	int len;
	INT_A buf[0];	
}INT_B;

typedef struct STR_LONG_A {
	long len;
	char buf[0];
}LONG_A;

typedef struct STR_LONG_B
{
	int len;
	LONG_A buf[0];
}LONG_B;

struct STR_LONG_A_ {
	long len;
	char buf[0];
}__attribute ((packed));
typedef struct STR_LONG_A_ LONG_A_;

struct STR_LONG_B_
{
	int len;
	LONG_A_ buf[0];
}__attribute ((packed));
typedef struct STR_LONG_B_ LONG_B_;

int main(int argc, char *argv[])
{
	printf("size of INT_A is:%d\n", sizeof(INT_A));
	printf("size of INT_B is:%d\n", sizeof(INT_B));
	printf("size of LONG_A is:%d\n", sizeof(LONG_A));
	printf("size of LONG_B is:%d\n", sizeof(LONG_B));
	
	printf("size of LONG_A_ is:%d\n", sizeof(LONG_A_));
	printf("size of LONG_B_ is:%d\n", sizeof(LONG_B_));
	
	INT_B *p_int_b = (INT_B*)malloc(1024);
	LONG_B *p_long_b = (LONG_B*)malloc(1024);
	
	printf("point of INT_B is:%p, point of INT_A is:%p\n", p_int_b, p_int_b->buf);
	printf("point of LONG_B is:%p, point of LONG_A is:%p\n", p_long_b, p_long_b->buf);
	
	return 0;
}
