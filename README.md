#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <termios.h>
#include <errno.h>
#include <time.h>
#include <linux/ioctl.h>
#include <linux/watchdog.h>
#include <arpa/inet.h>
#define FALSE -1
#define TRUE 0
#define uint16_t (unsigned short int)
#define uint32_t (unsigned int)

int fd;

int speed_arr[] = { B38400, B19200, B9600, B4800, B2400, B1200, B300,B38400, B19200, B9600, B4800, B2400, B1200, B300, };
int name_arr[] = {38400,  19200,  9600,  4800,  2400,  1200,  300, 38400, 19200,  9600, 4800, 2400, 1200,  300, };
/*
unsigned char req_vb[] = {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x2,0x0,0x14,0x0,0x1,0x84,0x0,0x0,0x0,0x7b,0x16};
*/
unsigned char req_vw2[] = {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x4,0x0,0x7,0x0,0x1,0x84,0x0,0x0,0x0,0x70,0x16};
unsigned char req_vd6[] = {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x6,0x0,0x2,0x0,0x1,0x84,0x0,0x0,0x30,0x9d,0x16};
unsigned char req_vd10[] =  {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x6,0x0,0x2,0x0,0x1,0x84,0x0,0x00,0x50,0xbd,0x16};
unsigned char req_vd14[] =  {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x6,0x0,0x2,0x0,0x1,0x84,0x0,0x0,0x70,0xdd,0x16};
unsigned char ack_ppi[] = {0x10,2,0,0x5C,0x5E,0x16};

float vd2f(unsigned char *src)
{
	float f;
	unsigned int i;

	i = ntohl((unsigned int)(src[1]*16777216+src[0]*65536+src[3]*256+src[2]));
//	i= src[2]*16777216+src[3]*65536+src[0]*256+src[1];

	memcpy(&f, &i, sizeof(float));

return f;
}

void set_speed(int fd, int speed){
    int   i;
    int   status;
    struct termios   Opt;
    tcgetattr(fd, &Opt);
    for ( i= 0;  i < sizeof(speed_arr) / sizeof(int);  i++) {
        if  (speed == name_arr[i]) {
            tcflush(fd, TCIOFLUSH);
            cfsetispeed(&Opt, speed_arr[i]);
            cfsetospeed(&Opt, speed_arr[i]);
            status = tcsetattr(fd, TCSANOW, &Opt);
            if  (status != 0) {
                perror("tcsetattr fd1");
                return;
            }
            tcflush(fd,TCIOFLUSH);
        }
    }
}
int set_Parity(int fd,int databits,int stopbits,int parity)
{
    struct termios options;
    if  ( tcgetattr( fd,&options)  !=  0)
    {
        perror("SetupSerial 1");
        return(FALSE);
    }
    options.c_cflag &= ~CSIZE;
    switch (databits)
    {
    case 7:
        options.c_cflag |= CS7;
        break;
    case 8:
        options.c_cflag |= CS8;
        break;
    default:
        fprintf(stderr,"Unsupported data size\n");
        return (FALSE);
    }
    switch (parity)
    {
    case 'n':
    case 'N':
        options.c_cflag &= ~PARENB;
        options.c_iflag &= ~INPCK;
        break;
    case 'o':
    case 'O':
        options.c_cflag |= (PARODD | PARENB);
        options.c_iflag |= INPCK;
        break;
    case 'e':
    case 'E':
        options.c_cflag |= PARENB;
        options.c_cflag &= ~PARODD;
        options.c_iflag |= INPCK;
        break;
    case 'S':
    case 's':
        options.c_cflag &= ~PARENB;
        options.c_cflag &= ~CSTOPB;
        break;
    default:
        fprintf(stderr,"Unsupported parity\n");
        return (FALSE);
    }
    switch (stopbits)
    {
    case 1:
        options.c_cflag &= ~CSTOPB;
        break;
    case 2:
        options.c_cflag |= CSTOPB;
        break;
    default:
        fprintf(stderr,"Unsupported stop bits\n");
        return (FALSE);
    }


    options.c_iflag &= ~(IGNBRK|BRKINT|PARMRK|ISTRIP|INLCR|IGNCR|ICRNL|IXON);
    options.c_oflag &= ~OPOST;
    options.c_lflag &= ~(ECHO|ECHONL|ICANON|ISIG|IEXTEN);

    /* Set input parity option */
    
    if (parity != 'n')
        options.c_iflag |= INPCK;
    options.c_cc[VTIME] = 150; // 15 seconds
    options.c_cc[VMIN] = 0;
    
    
    tcflush(fd,TCIFLUSH); /* Update the options and do it NOW */
    if (tcsetattr(fd,TCSANOW,&options) != 0)
    {
        perror("SetupSerial 3");
        return (FALSE);
    }
    return (TRUE);
}


unsigned char checksum(char *reqstr,int count)
{
    int i=0,j,sum=0;

    while(i<5){if(reqstr[i]==0x68) break;i++;}

    for(j=i;j<count;j++)
    {
        sum+=(unsigned char)reqstr[j];
        //printf("j=%d,%x ,sum %2x\n",j,reqstr[j],sum);
    }
    return (sum &0xff);
}

int readvb(int start,int len,unsigned char *myvb)
{
	int x;
	unsigned char blk[512];
	unsigned char req_str[] = {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x2,0x0,0x12,0x0,0x1,0x84,0x0,0x0,0x0,0x7b,0x16};

     req_str[22]=0x02; //vb:02 VW:04 VD:06...
     req_str[24]=len; //number of data
     req_str[29]=((start *8)>>8) & 0xff;  //offset high
     req_str[30]=(start*8)&0xff;  //offset low, number of bytes from  0 * 8
     int checksum=0;
     for(x=4;x<=30;x++) checksum+=req_str[x];
     req_str[31]=checksum & 0xFF;

//for(x=0;x<33;x++) printf(" %02X",req_str[x]&0xff);printf("\n");
//printf(" 0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2\n");
////////// ready /////////////
     int n = write(fd,req_str,33); //send req pack to PLC
//	printf("VBreq:%02d\n",n);

    usleep(1000*100);

    int nread = read(fd, blk, 1);
	if(nread<=0)
	{
		printf("error\n");return -1;
	}
	if ((unsigned char)blk[0]==0xE5)	// E5 PLC confirm got req pack
	{
		n = write(fd,ack_ppi,6);	// make suree do request command ...
//		printf("REQ has RECV, %02d\n",n);
	}
//    usleep(1000*100);	//wait more if get more data
    int hasread=0;
    while(hasread<27+len)
    {
        nread = read(fd, blk+hasread, 1);
	if(nread<=0)
	{
		printf("error\n");break;
	}
	hasread+=nread;
    }
//	for( x=0;x<hasread;x++) myvb[x]=blk[x];
//	printf("%d==",hasread);
//	printf(" 0 1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0 \n");
//	for(ix=0;x<hasread;x++) printf("%02X ",blk[x]&0xff);printf("\n");
	if ((blk[19]==04)&&(blk[20]==01)&&(blk[21]==0xff)&&(blk[22]==04))
	for( x=25;x<hasread-2;x++) {myvb[x-25]=blk[x];}

    usleep(1000*200);	//wait more if get more data
/*
unsigned char F[4];
F[0]=(unsigned char)blk[33];
F[1]=(unsigned char)blk[34];
F[2]=(unsigned char)blk[35];
F[3]=(unsigned char)blk[36];
printf("+++++++%02x %02x %02x %02x +++++++++++%f++++++++++++++++\n",F[0],F[1],F[2],F[3],vd2f((unsigned char*)F));
*/
}

int readvw(int start,int len,unsigned char *myvw)
{
	int x;
	unsigned char blk[512];
////////////////////////////////////////////////////////////////////////////////////////////////
	unsigned char req_str[] = {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x2,0x0,0x12,0x0,0x1,0x84,0x0,0x0,0x0,0x7b,0x16};

     req_str[22]=0x04; //vb:02 VW:04 VDW:06...
     req_str[24]=len; //number of data
     req_str[29]=((start *8)>>8) & 0xff;  //offset high
     req_str[30]=(start*8)&0xff;  //offset low, number of bytes from  0 * 8
     int checksum=0;
     for( x=4;x<=30;x++) checksum+=req_str[x];
     req_str[31]=checksum & 0xFF;

//for( x=0;x<33;x++) printf("%02X ",req_str[x]&0xff);printf("\n");
////////// ready /////////////
     int n = write(fd,req_str,33); //send req pack to PLC
//	printf("VWreq:%02d\n",n);

    usleep(1000*100);

    int nread = read(fd, blk, 1);
	if(nread<=0)
	{
		printf("error\n");return -1;
	}
	if ((unsigned char)blk[0]==0xE5)	// E5 PLC confirm got req pack
	{
		n = write(fd,ack_ppi,6);	// make suree do request command ...
//		printf("REQ has RECV, %02d\n",n);
	}


    int hasread=0;
    while(hasread<(27+len*2))
    {
        nread = read(fd, blk+hasread, 1);
	if(nread<=0)
	{
		printf("error\n");break;
	}
	hasread+=nread;
//for( x=0;x<hasread;x++) printf("%02X ",blk[x]&0xff);printf("\n");
    }
//	printf(" 0 1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0 \n");
//	for( x=0;x<hasread;x++) printf("%02X ",blk[x]&0xff);printf("\n");
	if ((blk[19]==04)&&(blk[20]==01)&&(blk[21]==0xff)&&(blk[22]==04))
	for( x=25;x<hasread-2;x++) {myvw[x-25]=blk[x];printf("%02x ",blk[x]);}


    usleep(1000*200);	//wait more if get more data
/*
unsigned char F[4];
F[0]=(unsigned char)blk[33];
F[1]=(unsigned char)blk[34];
F[2]=(unsigned char)blk[35];
F[3]=(unsigned char)blk[36];
printf("+++++++%02x %02x %02x %02x +++++++++++%f++++++++++++++++\n",F[0],F[1],F[2],F[3],vd2f((unsigned char*)F));
*/
}

int readvd(int start,int len,unsigned char *myvd)
{
	int x;
	unsigned char blk[512];
////////////////////////////////////////////////////////////////////////////////////////////////
	unsigned char req_str[] = {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x2,0x0,0x12,0x0,0x1,0x84,0x0,0x0,0x0,0x7b,0x16};

     req_str[22]=0x06; //vb:02 VW:04 VD:06...
     req_str[24]=len; //number of data
     req_str[29]=((start *8)>>8) & 0xff;  //offset high
     req_str[30]=(start*8)&0xff;  //offset low, number of bytes from  0 * 8
     int checksum=0;
     for(x=4;x<=30;x++) checksum+=req_str[x];
     req_str[31]=checksum & 0xFF;

//for( x=0;x<33;x++) printf("%02X ",req_str[x]&0xff);printf("\n");
////////// ready /////////////
     int n = write(fd,req_str,33); //send req pack to PLC
//	printf("VDreq:%02d\n",n);

    usleep(1000*100);

    int nread = read(fd, blk, 1);
	if(nread<=0)
	{
		printf("error\n");return -1;
	}
	if ((unsigned char)blk[0]==0xE5)	// 'E5' PLC confirm got req pack
	{
		n = write(fd,ack_ppi,6);	// make suree do request command ...
//		printf("REQ has RECV, %02d\n",n);
	}

     int hasread=0;
    while(hasread<(27+len*4))
    {
        nread = read(fd, blk+hasread, 1);
	if(nread<=0)
	{
		printf("error\n");break;
	}
	hasread+=nread;
//for( x=0;x<hasread;x++) printf("%02X ",blk[x]&0xff);printf("\n");
    }
//	printf(" 0 1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0 \n");
//	for( x=0;x<hasread;x++) printf("%02X ",blk[x]&0xff);printf("\n");
	if ((blk[19]==04)&&(blk[20]==01)&&(blk[21]==0xff)&&(blk[22]==04))
	for( x=25;x<hasread-2;x++) {myvd[x-25]=blk[x];printf("%02x ",blk[x]);}


    usleep(1000*100);	//wait more if get more data
/*
unsigned char F[4];
F[0]=(unsigned char)blk[33];
F[1]=(unsigned char)blk[34];
F[2]=(unsigned char)blk[35];
F[3]=(unsigned char)blk[36];
printf("+++++++%02x %02x %02x %02x +++++++++++%f++++++++++++++++\n",F[0],F[1],F[2],F[3],vd2f((unsigned char*)F));
*/
}

int readQ0(int start,int len,unsigned char *myQ0)
//void readvb()
{
	int x;
	unsigned char blk[512];
////////////////////////////////////////////////////////////////////////////////////////////////
	unsigned char req_str[] = {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x2,0x0,0x12,0x0,0x1,0x84,0x0,0x0,0x0,0x7b,0x16};
     req_str[22]=0x01; //Q0.1：01 vb:02 VW:04 VD:06... !!! 0x02for Q0.0 I0.0 return (len) bytes 0x01 onluy return 1 bit !!
     req_str[24]=len; //number of data
     req_str[26]=0;
     req_str[27]=0x82;
     req_str[29]=((start)>>8) & 0xff;  //offset high
     req_str[30]=(start)&0xff;  //offset low, number of bytes from  0 * 8
     int checksum=0;
     for( x=4;x<=30;x++) checksum+=req_str[x];
     req_str[31]=checksum & 0xFF;
////////// ready /////////////

//for(x=0;x<33;x++) printf("%02X ",req_str[x]&0xff);printf("\n");
////////// ready /////////////
     int n = write(fd,req_str,33); //send req pack to PLC
//	printf("Q0 req:%02d\n",n);

    usleep(1000*100);

    int nread = read(fd, blk, 1);
	if(nread<=0)
	{
		printf("error\n");return -1;
	}
	if ((unsigned char)blk[0]==0xE5)	// E5 PLC confirm got req pack
	{
		n = write(fd,ack_ppi,6);	// make suree do request command ...
//		printf("REQ has RECV, %02d\n",n);
	}
    usleep(1000*100);	//wait more if get more data

    nread = read(fd, blk, 27+len);
	if(nread!=27+len)
	{
		printf("error\n");
	}
//printf(" 0 1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0 \n");
//	for(x=0;x<nread;x++) printf("%02X ",blk[x]&0xff);printf("\n");

	if ((blk[19]==04)&&(blk[20]==01)&&(blk[21]==0xff)&&(blk[22]==03))
	for(x=25;x<nread-2;x++) {myQ0[x-25]=blk[x];}
//	printf("%d==",nread);
//	for( x=0;x<nread;x++) printf("%02X ",blk[x]&0xff);printf("\n");

}
int readbitI0(int start,int len,unsigned char *myI0)
//void readvb()
{
	int x;
	unsigned char blk[512];
////////////////////////////////////////////////////////////////////////////////////////////////
	unsigned char req_str[] = {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x1,0x0,0x12,0x0,0x1,0x84,0x0,0x0,0x0,0x7b,0x16};

     req_str[22]=0x01; //Q0.1：01 vb:02 VW:04 VD:06... !!! 0x02for Q0.0 I0.0 return (len) bytes 0x01 onluy return 1 bit !!
     req_str[24]=len; //number of data
     req_str[26]=0;
     req_str[27]=0x81;
     req_str[28]=((start)>>16) & 0xff;  //offset high
     req_str[29]=((start)>>8) & 0xff;  //offset high
     req_str[30]=(start)&0xff;  //offset low, number of bytes from  0 * 8
     int checksum=0;
     for( x=4;x<=30;x++) checksum+=req_str[x];
     req_str[31]=checksum & 0xFF;

////////// ready /////////////
//for( x=0;x<33;x++) printf(" %02X",req_str[x]&0xff);printf("\n");
//printf(" 0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2\n");////////// ready /////////////
     int n = write(fd,req_str,33); //send req pack to PLC
//	printf("I0 req:%02d\n",n);

    usleep(1000*100);

    int nread = read(fd, blk, 1);
	if(nread<=0)
	{
		printf("error\n");return -1;
	}
	if ((unsigned char)blk[0]==0xE5)	// E5 PLC confirm got req pack
	{
		n = write(fd,ack_ppi,6);	// make suree do request command ...
//		printf("REQ has RECV, %02d\n",n);
	}
    usleep(1000*100);	//wait more if get more data

    nread = read(fd, blk, 27+len);
	if(nread!=27+len)
	{
		printf("error\n");
	}
//printf(" 0 1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0 \n");
//	for( x=0;x<nread;x++) printf("%02X ",blk[x]&0xff);printf("\n");

	if ((blk[19]==04)&&(blk[20]==01)&&(blk[21]==0xff)&&(blk[22]==03))
	for(x=25;x<nread-2;x++) {myI0[x-25]=blk[x];}
}

int readbyteI0(int start,int len,unsigned char *myI0)
//void readvb()
{
	int x;
	unsigned char blk[512];
////////////////////////////////////////////////////////////////////////////////////////////////
	unsigned char req_str[] = {0x68,0x1B,0x1B,0x68,0x2,0x0,0x6C,
			0x32,0x1,0x0,0x0,0x0,0x0,0x0,
			0xE,0x0,0x0,0x4,0x1,0x12,0xA,0x10,0x1,0x0,0x12,0x0,0x1,0x84,0x0,0x0,0x0,0x7b,0x16};

     req_str[22]=0x02; //Q0.1：01 vb:02 VW:04 VD:06... !!! 0x02for Q0.0 I0.0 return (len) bytes 0x01 onluy return 1 bit !!
     req_str[24]=len; //number of data
     req_str[26]=0;
     req_str[27]=0x81;
     req_str[28]=((start)>>16) & 0xff;  //offset high
     req_str[29]=((start)>>8) & 0xff;  //offset high
     req_str[30]=(start)&0xff;  //offset low, number of bytes from  0 * 8
     int checksum=0;
     for( x=4;x<=30;x++) checksum+=req_str[x];
     req_str[31]=checksum & 0xFF;

////////// ready /////////////
//for(x=0;x<33;x++) printf(" %02X",req_str[x]&0xff);printf("\n");
//printf(" 0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2\n");////////// ready /////////////
     int n = write(fd,req_str,33); //send req pack to PLC
//	printf("I0 req:%02d\n",n);

    usleep(1000*100);

    int nread = read(fd, blk, 1);
	if(nread<=0)
	{
		printf("error\n");return -1;
	}
	if ((unsigned char)blk[0]==0xE5)	// E5 PLC confirm got req pack
	{
		n = write(fd,ack_ppi,6);	// make suree do request command ...
//		printf("REQ has RECV, %02d\n",n);
	}
    usleep(1000*100);	//wait more if get more data

    nread = read(fd, blk, 27+len);
	if(nread!=27+len)
	{
		printf("error\n");
	}
//printf(" 0 1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5  6  7  8  9  0 \n");
//	for( x=0;x<nread;x++) printf("%02X ",blk[x]&0xff);printf("\n");

	if ((blk[19]==04)&&(blk[20]==01)&&(blk[21]==0xff)&&(blk[22]==04))
	for( x=25;x<nread-2;x++) {myI0[x-25]=blk[x];}

//	printf("%d==",nread);
//	for( x=0;x<nread;x++) printf("%02X ",blk[x]&0xff);printf("\n");
}
short int a2i16(unsigned char*s)
{
	return((s[0]<<8)+s[1]);
}
int a2i32(unsigned char*s)
{
	return((s[0]<<24)+(s[1]<<16)+(s[2]<<8)+s[3]);
}
/*int a2i32(unsigned char*s)
{
	unsigned int tt;
	tt =((unsigned char)s[0]<<24) + ((unsigned char)s[1]<<16) + ((unsigned char)s[2]<<8) + (unsigned char)s[3];
	return tt;
}
*/
float a2f32(unsigned char*s)
{
	union{
		unsigned int i;
		float f;
		unsigned char c[4];
	}a2f;
/*
	a2f.c[0]=s[3];
	a2f.c[1]=s[2];
	a2f.c[2]=s[1];
	a2f.c[3]=s[0];
*/
	a2f.i = ntohl((unsigned int)(s[3]*16777216+s[2]*65536+s[1]*256+s[0]));

	return a2f.f;
}

int main()
{printf("This program updates last time at %s   %s\n",__TIME__,__DATE__);

    union {
	short int si[2];
	int i;
	unsigned char c[4];
    }c2i;

    FILE *fp;
    int x,n,nread=0;
    unsigned char buff[512],F[4];
/*    unsigned char req[] = {0x68,0x1b,0x1b,0x68,0x02,0x00,0x6c,
			0x32,0x01,0x0,0x0,0x0,0x0,0x0,
			0x4,0x1,0x12,0x0a,0x10,
			0x02,0x00,0x08,0x00,0x00,0x03,0x0,0x05,0xE0,0xD2,0x16};
*/

    fd = open("/dev/ttyUSB0",O_RDWR);
//    fd = open("/dev/ttyO2",O_RDWR);
    if(fd == -1){perror("open serialport error\n");exit(-1);}
    else{	printf("open ");printf("%s",ttyname(fd));printf(" succesfully\n");}

    set_speed(fd,9600);
    if (set_Parity(fd,8,1,'E') == FALSE)  {	printf("Set Parity Error\n");exit (0);}



int loop=0;
while(loop++<1000)
{
readvb(0,24,(unsigned char*)buff);for(x=0;x<24;x++) printf("%02X ",buff[x]&0xff);printf("\n");

short int x=a2i16(buff);
int y=a2i32(buff+20);
float r0=a2f32(buff+8);
float r1=a2f32(buff+12);
float r2=a2f32(buff+16);

//printf("x=%d,y=%d,r0=%f,r1=%f,r2=%f\n",x,y,r0,r1,r2);
printf("外触发计数：%d,32位自动计数：%d,正玄波度：%0.0f,度*PI/180：%f,正玄值：%f\n",x,y,r0,r1,r2);
/*
unsigned char F[4];
F[0]=(unsigned char)buff[10];
F[1]=(unsigned char)buff[11];
F[2]=(unsigned char)buff[8];
F[3]=(unsigned char)buff[9];
printf("+++++++%02x %02x %02x %02x +++++++++++%f++++++++++++++++\n",F[0],F[1],F[2],F[3],vd2f((unsigned char*)F));

*/

/*


readvb(2,4,(unsigned char*)buff);c2i.c[0]=buff[3];c2i.c[1]=buff[2];c2i.c[2]=buff[1];c2i.c[3]=buff[0];printf("	vb=%d\n",c2i.i);
readvw(2,2,(unsigned char*)buff);c2i.c[0]=buff[3];c2i.c[1]=buff[2];c2i.c[2]=buff[1];c2i.c[3]=buff[0];printf("	vw=%d\n",c2i.i);
readvd(20,1,(unsigned char*)buff);c2i.c[0]=buff[3];c2i.c[1]=buff[2];c2i.c[2]=buff[1];c2i.c[3]=buff[0];printf("	vd20=%d\n",c2i.i);
*/
readQ0(2,1,(unsigned char*)buff);printf("继电器 Q2.0 bit %02x\n",buff[0]);
/*
readbitI0(1,1,(unsigned char*)buff);printf("I0.1 bit %02x\n",buff[0]);
readbitI0(2,1,(unsigned char*)buff);printf("I0.2 bit %02x\n",buff[0]);
readbyteI0(0,4,(unsigned char*)buff);printf("I0 Byte %02x %02x %02x %02x\n",buff[0],buff[1],buff[2],buff[3]);
printf("\n");
*/
usleep(1000000);
}
////////////////////////////////////////////////////////////////////////////////////////////////
    close(fd);
    return 0;
}
