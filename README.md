# minix-driver
Some Minix drivers.

# Hello Driver

## hello.c
```
#include <minix/drivers.h>
#include <minix/chardriver.h>
#include <stdio.h>
#include <stdlib.h>
#include <minix/ds.h>
#include <string.h>
#include "hello.h"

/*
 * Function prototypes for the hello driver.
 */
static int hello_open(message *m);
static int hello_close(message *m);
static struct device * hello_prepare(dev_t device);
static int hello_transfer(endpoint_t endpt, int opcode, u64_t position,
	iovec_t *iov, unsigned int nr_req, endpoint_t user_endpt, unsigned int
	flags);

/* SEF functions and variables. */
static void sef_local_startup(void);
static int sef_cb_init(int type, sef_init_info_t *info);
static int sef_cb_lu_state_save(int);
static int lu_state_restore(void);

/* Entry points to the hello driver. */
static struct chardriver hello_tab =
{
    hello_open,
    hello_close,
    nop_ioctl,
    hello_prepare,
    hello_transfer,
    nop_cleanup,
    nop_alarm,
    nop_cancel,
    nop_select,
    NULL
};

/** Represents the /dev/hello device. */
static struct device hello_device;

/** State variable to count the number of times the device has been opened. */
static int open_counter;
static char helloBuffer[MAX_BUFFER];

static int hello_open(message *UNUSED(m))
{
    printf("hello_open(). Called %d time(s).\n", ++open_counter);
    strcpy(helloBuffer, HELLO_MESSAGE);
    return OK;
}

static int hello_close(message *UNUSED(m))
{
    printf("hello_close()\n");
    return OK;
}

static struct device * hello_prepare(dev_t UNUSED(dev))
{
    hello_device.dv_base = make64(0, 0);
    hello_device.dv_size = make64(strlen(HELLO_MESSAGE), 0);
    return &hello_device;
}

static int hello_transfer(endpoint_t endpt, int opcode, u64_t position,
    iovec_t *iov, unsigned nr_req, endpoint_t UNUSED(user_endpt),
    unsigned int UNUSED(flags))
{
    int bytes, ret;
    char readBuffer[WRITE_SIZE];

    printf("hello_transfer()\n");

    if (nr_req != 1)
    {
        /* This should never trigger for character drivers at the moment. */
        printf("HELLO: vectored transfer request, using first element only\n");
    }

    bytes = strlen(HELLO_MESSAGE) - ex64lo(position) < iov->iov_size ?
            strlen(HELLO_MESSAGE) - ex64lo(position) : iov->iov_size;

    if (bytes <= 0)
    {
        return OK;
    }
    switch (opcode)
    {
        case DEV_GATHER_S:
            ret = sys_safecopyto(endpt, (cp_grant_id_t) iov->iov_addr, 0,
                                (vir_bytes) (HELLO_MESSAGE + ex64lo(position)),
                                 bytes);
            iov->iov_size -= bytes;
            break;
        case DEV_SCATTER_S:
            ret = sys_safecopyfrom(endpt, (cp_grant_id_t) iov->iov_addr, 0, (vir_bytes)readBuffer, WRITE_SIZE);
            readBuffer[WRITE_SIZE-1]='\0';
            printf("Read buffer: %s\n", readBuffer);
            strlcat(helloBuffer, readBuffer, MAX_BUFFER);
            helloBuffer[MAX_BUFFER-1]='\0';
            printf("Hello buffer: %s\n", helloBuffer);
            break;
        default:
            return EINVAL;
    }
    return ret;
}

static int sef_cb_lu_state_save(int UNUSED(state)) {
/* Save the state. */
    ds_publish_u32("open_counter", open_counter, DSF_OVERWRITE);

    return OK;
}

static int lu_state_restore() {
/* Restore the state. */
    u32_t value;

    ds_retrieve_u32("open_counter", &value);
    ds_delete_u32("open_counter");
    open_counter = (int) value;

    return OK;
}

static void sef_local_startup()
{
    /*
     * Register init callbacks. Use the same function for all event types
     */
    sef_setcb_init_fresh(sef_cb_init);
    sef_setcb_init_lu(sef_cb_init);
    sef_setcb_init_restart(sef_cb_init);

    /*
     * Register live update callbacks.
     */
    /* - Agree to update immediately when LU is requested in a valid state. */
    sef_setcb_lu_prepare(sef_cb_lu_prepare_always_ready);
    /* - Support live update starting from any standard state. */
    sef_setcb_lu_state_isvalid(sef_cb_lu_state_isvalid_standard);
    /* - Register a custom routine to save the state. */
    sef_setcb_lu_state_save(sef_cb_lu_state_save);

    /* Let SEF perform startup. */
    sef_startup();
}

static int sef_cb_init(int type, sef_init_info_t *UNUSED(info))
{
/* Initialize the hello driver. */
    int do_announce_driver = TRUE;

    open_counter = 0;
    switch(type) {
        case SEF_INIT_FRESH:
            printf("%s", HELLO_MESSAGE);
        break;

        case SEF_INIT_LU:
            /* Restore the state. */
            lu_state_restore();
            do_announce_driver = FALSE;

            printf("%sHey, I'm a new version!\n", HELLO_MESSAGE);
        break;

        case SEF_INIT_RESTART:
            printf("%sHey, I've just been restarted!\n", HELLO_MESSAGE);
        break;
    }

    /* Announce we are up when necessary. */
    if (do_announce_driver) {
        chardriver_announce();
    }

    /* Initialization completed successfully. */
    return OK;
}

int main(void)
{
    /*
     * Perform initialization.
     */
    sef_local_startup();

    /*
     * Run the main loop.
     */
    chardriver_task(&hello_tab, CHARDRIVER_SYNC);
    return OK;
}
```

## hello.h
```
#ifndef __HELLO_H
#define __HELLO_H

/** The Hello, World! message. */
#define HELLO_MESSAGE "Hello"
#define WRITE_SIZE 10
#define MAX_BUFFER 30

#endif /* __HELLO_H */
```

## testHello.c
```
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include "hello.h"

int main(void) {
    int fp;
    ssize_t bytes;
    char hello_message[WRITE_SIZE];
    strncpy(hello_message, "cutoff word", WRITE_SIZE);

    // Attempt to read two full messages.
    printf("OPEN\n");
    fp = open("/dev/hello", O_RDWR);
    printf("File descriptor: %d\n", fp);
    printf("Writing to file.\n");
    bytes = write(fp, hello_message, WRITE_SIZE);   
    printf("Bytes written: %d\n", bytes);
    bytes = write(fp, hello_message, WRITE_SIZE);   
    printf("Bytes written: %d\n", bytes);
    bytes = write(fp, hello_message, WRITE_SIZE);   
    printf("Bytes written: %d\n", bytes);
    close(fp);
    printf("CLOSE\n\n");
}
```

# Magic8 Driver

## magic8.c
```
#include <minix/drivers.h>
#include <minix/chardriver.h>
#include <minix/ioctl.h>
#include <stdio.h>
#include <stdlib.h>
#include <minix/ds.h>
#include <stdlib.h>
#include "magic8.h"

/*
 * Function prototypes for the magic driver.
 */
static int magic_open(message *m);
static int magic_close(message *m);
static struct device * magic_prepare(dev_t device);
static int magic_transfer(endpoint_t endpt, int opcode, u64_t position,
	iovec_t *iov, unsigned int nr_req, endpoint_t user_endpt, unsigned int
	flags);
static int magic_ioctl(message *m);

/* SEF functions and variables. */
static void sef_local_startup(void);
static int sef_cb_init(int type, sef_init_info_t *info);
static int sef_cb_lu_state_save(int);
static int lu_state_restore(void);
static int yesno;
static int msgCase = DEFAULTMAGIC8;

/* Entry points to the magic driver. */
static struct chardriver magic_tab =
{
    magic_open,
    magic_close,
    magic_ioctl,
    magic_prepare,
    magic_transfer,
    nop_cleanup,
    nop_alarm,
    nop_cancel,
    nop_select,
    NULL
};

/** Represents the /dev/magic device. */
static struct device magic_device;

/** State variable to count the number of times the device has been opened. */
static int open_counter;

static int magic_open(message *UNUSED(m))
{
    yesno = random();
    printf("Random number generated: %d\n", yesno);
    printf("magic_open(). Called %d time(s).\n", ++open_counter);
    return OK;
}

static int magic_close(message *UNUSED(m))
{
    printf("magic_close()\n");
    return OK;
}

static struct device * magic_prepare(dev_t UNUSED(dev))
{
    magic_device.dv_base = make64(0, 0);
    magic_device.dv_size = make64(strlen(MAGIC_MESSAGE), 0);
    return &magic_device;
}

static int magic_ioctl(message *m_ptr){
    printf("magic_ioctl()\n");
    printf("Message source: %d\n", m_ptr->m_source);
    printf("Message type: %d\n", m_ptr->m_type);
    printf("Message one: %d\n", m_ptr->m_u.m_m1.m1i1);
    printf("Message: %d\n", m_ptr->COUNT);
    switch(m_ptr->COUNT) {
        case LOWERMAGIC8:
            msgCase = LOWERMAGIC8;
            break;
        case UPPERMAGIC8:
            msgCase = UPPERMAGIC8;
            break;
        default:
            msgCase = DEFAULTMAGIC8;
            break;
    }
    return 0;
}

static int magic_transfer(endpoint_t endpt, int opcode, u64_t position,
    iovec_t *iov, unsigned nr_req, endpoint_t UNUSED(user_endpt),
    unsigned int UNUSED(flags))
{
    int bytes, ret;
    char magic_message[4];

    // Convert to 1/0 (true/false, yes/no).
    yesno%=2;
    switch(msgCase) {
        case LOWERMAGIC8:
            memcpy(magic_message, (yesno) ? MAGIC_YES_LOWER : MAGIC_NO_LOWER, sizeof magic_message);
            break;
        case UPPERMAGIC8:
            memcpy(magic_message, (yesno) ? MAGIC_YES_UPPER : MAGIC_NO_UPPER, sizeof magic_message);
            break;
        default:
            memcpy(magic_message, (yesno) ? MAGIC_YES : MAGIC_NO, sizeof magic_message);
            break;
    }
    msgCase=DEFAULTMAGIC8;


    printf("magic_transfer()\n");
    printf("Results in %s\n", magic_message);


    if (nr_req != 1)
    {
        /* This should never trigger for character drivers at the moment. */
        printf("magic: vectored transfer request, using first element only\n");
    }

    bytes = strlen(magic_message) - ex64lo(position) < iov->iov_size ?
            strlen(magic_message) - ex64lo(position) : iov->iov_size;

    if (bytes <= 0)
    {
        return OK;
    }
    switch (opcode)
    {
        case DEV_GATHER_S:
            ret = sys_safecopyto(endpt, (cp_grant_id_t) iov->iov_addr, 0,
                                (vir_bytes) (magic_message + ex64lo(position)),
                                 bytes);
            iov->iov_size -= bytes;
            break;

        default:
            return EINVAL;
    }
    return ret;
}

static int sef_cb_lu_state_save(int UNUSED(state)) {
/* Save the state. */
    ds_publish_u32("open_counter", open_counter, DSF_OVERWRITE);

    return OK;
}

static int lu_state_restore() {
/* Restore the state. */
    u32_t value;

    ds_retrieve_u32("open_counter", &value);
    ds_delete_u32("open_counter");
    open_counter = (int) value;

    return OK;
}

static void sef_local_startup()
{
    /*
     * Register init callbacks. Use the same function for all event types
     */
    sef_setcb_init_fresh(sef_cb_init);
    sef_setcb_init_lu(sef_cb_init);
    sef_setcb_init_restart(sef_cb_init);

    /*
     * Register live update callbacks.
     */
    /* - Agree to update immediately when LU is requested in a valid state. */
    sef_setcb_lu_prepare(sef_cb_lu_prepare_always_ready);
    /* - Support live update starting from any standard state. */
    sef_setcb_lu_state_isvalid(sef_cb_lu_state_isvalid_standard);
    /* - Register a custom routine to save the state. */
    sef_setcb_lu_state_save(sef_cb_lu_state_save);

    /* Let SEF perform startup. */
    sef_startup();
}

static int sef_cb_init(int type, sef_init_info_t *UNUSED(info))
{
/* Initialize the magic driver. */
    int do_announce_driver = TRUE;

    open_counter = 0;
    switch(type) {
        case SEF_INIT_FRESH:
            printf("%s", "I've been started!");
        break;

        case SEF_INIT_LU:
            /* Restore the state. */
            lu_state_restore();
            do_announce_driver = FALSE;

            printf("Hey, I'm a new version!\n");
        break;

        case SEF_INIT_RESTART:
            printf("Hey, I've just been restarted!\n");
        break;
    }

    /* Announce we are up when necessary. */
    if (do_announce_driver) {
        chardriver_announce();
    }

    /* Initialization completed successfully. */
    return OK;
}

int main(void)
{
    /*
     * Perform initialization.
     */
    sef_local_startup();

    /*
     * Run the main loop.
     */
    chardriver_task(&magic_tab, CHARDRIVER_SYNC);
    return OK;
}
```

## magic8.h
```
#ifndef __MAGIC_H
#define __MAGIC_H

/** Magic message. */
#define MAGIC_MESSAGE "Magic response.\n"
#define MAGIC_YES "Yes"
#define MAGIC_NO  "No "
#define MAGIC_YES_UPPER "YES"
#define MAGIC_NO_UPPER  "NO "
#define MAGIC_YES_LOWER "yes"
#define MAGIC_NO_LOWER  "no "

#define DEFAULTMAGIC8 0
#define UPPERMAGIC8 1
#define LOWERMAGIC8 2

#endif /* __MAGIC_H */
```

## testMagic8.c
```
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include "magic8.h"

int main(void) {
//    FILE *fp;
    int fp;
    ssize_t bytes;
    char magic_message[4] = "";
    int test=33;

    // Attempt to read two full messages.
    printf("OPEN\n");
    fp = open("/dev/magic8", O_RDONLY);
    printf("File descriptor: %d\n", fp);
    bytes = read(fp, magic_message, sizeof magic_message);
    printf("Bytes read: %d\n", bytes);
    bytes = read(fp, magic_message, sizeof magic_message);
    printf("Bytes read: %d\n", bytes);
    printf("Magic message: %s\n", magic_message);
    close(fp);
    printf("CLOSE\n\n");

    // Read a byte at a time.
    printf("OPEN\n");
    fp = open("/dev/magic8", O_RDONLY);
    printf("File descriptor: %d\n", fp);
    while((bytes = read(fp, magic_message, 1))) {
        printf("Read %d byte.\n", bytes);
        printf("%c\n", magic_message[0]);
    }
    close(fp);
    printf("CLOSE\n\n");

    // A read with UPPERMAGIC8 passed in.
    printf("OPEN\n");
    fp = open("/dev/magic8", O_RDONLY);
    printf("Sending UPPERMAGIC8.\n");
    ioctl(fp, UPPERMAGIC8, (void *)NULL);
    bytes = read(fp, magic_message, sizeof magic_message);
    printf("Magic message: %s\n", magic_message);
    close(fp);
    printf("CLOSE\n\n");

    // A read with LOWERMAGIC8 passed in.
    printf("OPEN\n");
    fp = open("/dev/magic8", O_RDONLY);
    printf("Sending LOWERMAGIC8.\n");
    ioctl(fp, LOWERMAGIC8, (void *)NULL);
    bytes = read(fp, magic_message, sizeof magic_message);
    printf("Magic message: %s\n", magic_message);
    close(fp);
    printf("CLOSE\n\n");
    
    // A read with no IOCTL message.
    printf("OPEN\n");
    fp = open("/dev/magic8", O_RDONLY);
    bytes = read(fp, magic_message, sizeof magic_message);
    printf("Magic message: %s\n", magic_message);
    close(fp);
    printf("CLOSE\n\n");
}
```
