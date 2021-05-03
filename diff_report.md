diff --git a/Makefile b/Makefile
index 6483959..6a982f7 100644
--- a/Makefile
+++ b/Makefile
@@ -1,7 +1,7 @@
 # Set flag to correct CS333 project number: 1, 2, ...
 # 0 == original xv6-pdx distribution functionality
-CS333_PROJECT ?= 0
-PRINT_SYSCALLS ?= 0
+CS333_PROJECT ?= 1
+PRINT_SYSCALLS ?= 1
 CS333_CFLAGS ?= -DPDX_XV6
 ifeq ($(CS333_CFLAGS), -DPDX_XV6)
 CS333_UPROGS +=        _halt _uptime
@@ -13,7 +13,7 @@ endif

 ifeq ($(CS333_PROJECT), 1)
 CS333_CFLAGS += -DCS333_P1
-CS333_UPROGS += #_date
+CS333_UPROGS += _date
 endif

 ifeq ($(CS333_PROJECT), 2)
@@ -123,7 +123,7 @@ AS = $(TOOLPREFIX)gas
 LD = $(TOOLPREFIX)ld
 OBJCOPY = $(TOOLPREFIX)objcopy
 OBJDUMP = $(TOOLPREFIX)objdump
-CFLAGS = -fno-pic -static -fno-builtin -fno-strict-aliasing -Og -Wall -MD -ggdb -m32 -Werror -fno-omit-frame-pointer -fno-aggressive-loop-optimizations
+CFLAGS = -fno-pic -static -fno-builtin -Werror -fno-strict-aliasing -Og -Wall -MD -ggdb -m32 -fno-omit-frame-pointer -fno-aggressive-loop-optimizations
 CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
 CFLAGS += $(CS333_CFLAGS)
 ASFLAGS = -m32 -gdwarf-2 -Wa,-divide
diff --git a/proc.c b/proc.c
index d030537..20f8d39 100644
--- a/proc.c
+++ b/proc.c
@@ -34,7 +34,7 @@ static struct {
 // list management function prototypes
 #ifdef CS333_P3
 static void initProcessLists(void);
-static void initFreeList(void);
+static void initFreeist(void);
 static void stateListAdd(struct ptrs*, struct proc*);
 static int  stateListRemove(struct ptrs*, struct proc* p);
 static void assertState(struct proc*, enum procstate, const char *, int);
@@ -149,6 +149,8 @@ allocproc(void)
   memset(p->context, 0, sizeof *p->context);
   p->context->eip = (uint)forkret;

+  p->start_ticks = ticks;
+
   return p;
 }

@@ -563,7 +565,7 @@ procdumpP2P3P4(struct proc *p, char *state_string)
 void
 procdumpP1(struct proc *p, char *state_string)
 {
-  cprintf("TODO for Project 1, delete this line and implement procdumpP1() in proc.c to print a row\n");
+  cprintf("%d\t%s\t\t%d\t%s\t%d\t", p->pid, p->name, p->start_ticks, states[p->state], p->sz);
   return;
 }
 #endif
@@ -914,4 +916,3 @@ checkProcs(const char *file, const char *func, int line)
   }
 }
 #endif // DEBUG
-
diff --git a/proc.h b/proc.h
index 0a0b4c5..0f67685 100644
--- a/proc.h
+++ b/proc.h
@@ -48,7 +48,8 @@ struct proc {
   int killed;                  // If non-zero, have been killed
   struct file *ofile[NOFILE];  // Open files
   struct inode *cwd;           // Current directory
-  char name[16];               // Process name (debugging)
+  char name[16];
+  uint start_ticks;               // Process name (debugging)
 };

 // Process memory is laid out contiguously, low addresses first:
diff --git a/syscall.c b/syscall.c
index 9105b52..20cc2c1 100644
--- a/syscall.c
+++ b/syscall.c
@@ -106,6 +106,9 @@ extern int sys_uptime(void);
 #ifdef PDX_XV6
 extern int sys_halt(void);
 #endif // PDX_XV6
+#ifdef CS333_P1
+extern int sys_date(void);
+#endif

 static int (*syscalls[])(void) = {
 [SYS_fork]    sys_fork,
@@ -132,6 +135,9 @@ static int (*syscalls[])(void) = {
 #ifdef PDX_XV6
 [SYS_halt]    sys_halt,
 #endif // PDX_XV6
+#ifdef CS333_P1
+[SYS_date]    sys_date,
+#endif // CS333_P1
 };

 #ifdef PRINT_SYSCALLS
@@ -172,6 +178,9 @@ syscall(void)
   num = curproc->tf->eax;
   if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
     curproc->tf->eax = syscalls[num]();
+    #ifdef PRINT_SYSCALLS
+    cprintf("%s -> %d \n", syscallnames[num], num);
+    #endif
   } else {
diff --git a/syscall.c b/syscall.c
index 9105b52..20cc2c1 100644
--- a/syscall.c
+++ b/syscall.c
@@ -106,6 +106,9 @@ extern int sys_uptime(void);
 #ifdef PDX_XV6
 extern int sys_halt(void);
 #endif // PDX_XV6
+#ifdef CS333_P1
+extern int sys_date(void);
+#endif

 static int (*syscalls[])(void) = {
 [SYS_fork]    sys_fork,
@@ -132,6 +135,9 @@ static int (*syscalls[])(void) = {
 #ifdef PDX_XV6
 [SS_halt]    sys_halt,
 #endif // PDX_XV6
+#ifdef CS333_P1
+[SYS_date]    sys_date,
+#endif // CS333_P1
 };

 #ifdef PRINT_SYSCALLS
@@ -172,6 +178,9 @@ syscall(void)
   num = curproc->tf->eax;
   if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
     curproc->tf->eax = syscalls[num]();
+    #ifdef PRINT_SYSCALLS
+    cprintf("%s -> %d \n", syscallnames[num], num);
+    #endif
   } else {
   } else {
     cprintf("%d %s: unknown sys call %d\n",
             curproc->pid, curproc->name, num);
diff --git a/syscall.h b/syscall.h
index 7fc8ce1..14f3e17 100644
--- a/syscall.h
+++ b/syscall.h
@@ -22,3 +22,4 @@
 #define SYS_close   SYS_mkdir+1
 #define SYS_halt    SYS_close+1
 // student system calls begin here. Follow the existing pattern.
+#define SYS_date    SYS_halt+1
diff --git a/sysproc.c b/sysproc.c
index 98563ea..8447d97 100644
--- a/sysproc.c
+++ b/sysproc.c
@@ -97,3 +97,17 @@ sys_halt(void)
   return 0;
 }
 #endif // PDX_XV6
+int
+sys_date(void)
+{
+  struct rtcdate *d;
+
+  if(argptr(0, (void*)&d, sizeof(struct rtcdate)) < 0)
+  {
+    return -1;
+  }
+
+  cmostime(d);
+
+  return 0;
+}
diff --git a/user.h b/user.h
index 31d9134..16d054b 100644
--- a/user.h
+++ b/user.h
@@ -43,3 +43,6 @@ int atoi(const char*);
 int atoo(const char*);
 int strncmp(const char*, const char*, uint);
 #endif // PDX_XV6
+#ifdef CS333_P1
+int date(struct rtcdate*);
+#endif // CS333_P1
diff --git a/usys.S b/usys.S
index 0d4eaed..84bd80b 100644
--- a/usys.S
+++ b/usys.S
@@ -30,3 +30,4 @@ SYSCALL(sbrk)
 SYSCALL(sleep)
 SYSCALL(uptime)
 SYSCALL(halt)
+SYSCALL(date)
diff --git a/Makefile b/Makefile
index 6483959..6a982f7 100644
--- a/Makefile
+++ b/Makefile
@@ -1,7 +1,7 @@
 # Set flag to correct CS333 project number: 1, 2, ...
 # 0 == original xv6-pdx distribution functionality
-CS333_PROJECT ?= 0
-PRINT_SYSCALLS ?= 0
+CS333_PROJECT ?= 1
+PRINT_SYSCALLS ?= 1
 CS333_CFLAGS ?= -DPDX_XV6
 ifeq ($(CS333_CFLAGS), -DPDX_XV6)
 CS333_UPROGS +=        _halt _uptime
@@ -13,7 +13,7 @@ endif

 ifeq ($(CS333_PROJECT), 1)
 CS333_CFLAGS += -DCS333_P1
-CS333_UPROGS += #_date
+CS333_UPROGS += _date
 endif

 ifeq ($(CS333_PROJECT), 2)
@@ -123,7 +123,7 @@ AS = $(TOOLPREFIX)gas
 LD = $(TOOLPREFIX)ld
 OBJCOPY = $(TOOLPREFIX)objcopy
 OBJDUMP = $(TOOLPREFIX)objdump
-CFLAGS = -fno-pic -static -fno-builtin -fno-strict-aliasing -Og -Wall -MD -
ggdb -m32 -Werror -fno-omit-frame-pointer -fno-aggressive-loop-optimizations
+CFLAGS = -fno-pic -static -fno-builtin -Werror -fno-strict-aliasing -Og -Wa
ll -MD -ggdb -m32 -fno-omit-frame-pointer -fno-aggressive-loop-optimizations
 CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2
>&1 && echo -fno-stack-protector)
 CFLAGS += $(CS333_CFLAGS)
 ASFLAGS = -m32 -gdwarf-2 -Wa,-divide
diff --git a/proc.c b/proc.c
index d030537..20f8d39 100644
--- a/proc.c
+++ b/proc.c
@@ -34,7 +34,7 @@ static struct {
 // list management function prototypes
 #ifdef CS333_P3
 static void initProcessLists(void);
-static void initFreeList(void);
+static void initFreeist(void);
 static void stateListAdd(struct ptrs*, struct proc*);
 static int  stateListRemove(struct ptrs*, struct proc* p);
 static void assertState(struct proc*, enum procstate, const char *, int);
@@ -149,6 +149,8 @@ allocproc(void)
   memset(p->context, 0, sizeof *p->context);
   p->context->eip = (uint)forkret;

+  p->start_ticks = ticks;
+
   return p;
 }

@@ -563,7 +565,7 @@ procdumpP2P3P4(struct proc *p, char *state_string)
 void
 procdumpP1(struct proc *p, char *state_string)
 {
-  cprintf("TODO for Project 1, delete this line and implement procdumpP1() in proc.c to print a row\n");
+  cprintf("%d\t%s\t\t%d\t%s\t%d\t", p->pid, p->name, p->start_ticks, states[p->state], p->sz);
   return;
 }
 #endif
@@ -914,4 +916,3 @@ checkProcs(const char *file, const char *func, int line)
   }
 }
 #endif // DEBUG
-
diff --git a/proc.h b/proc.h
index 0a0b4c5..0f67685 100644
--- a/proc.h
+++ b/proc.h
@@ -48,7 +48,8 @@ struct proc {
   int killed;                  // If non-zero, have been killed
   struct file *ofile[NOFILE];  // Open files
   struct inode *cwd;           // Current directory
-  char name[16];               // Process name (debugging)
+  char name[16];
+  uint start_ticks;               // Process name (debugging)
 };

 // Process memory is laid out contiguously, low addresses first:
diff --git a/syscall.c b/syscall.c
index 9105b52..20cc2c1 100644
--- a/syscall.c
+++ b/syscall.c
@@ -106,6 +106,9 @@ extern int sys_uptime(void);
 #ifdef PDX_XV6
 extern int sys_halt(void);
 #endif // PDX_XV6
+#ifdef CS333_P1
+extern int sys_date(void);
+#endif

 static int (*syscalls[])(void) = {
 [SYS_fork]    sys_fork,
@@ -132,6 +135,9 @@ static int (*syscalls[])(void) = {
 #ifdef PDX_XV6
 [SYS_halt]    sys_halt,
 #endif // PDX_XV6
+#ifdef CS333_P1
+[SYS_date]    sys_date,
+#endif // CS333_P1
 };

 #ifdef PRINT_SYSCALLS
@@ -172,6 +178,9 @@ syscall(void)
   num = curproc->tf->eax;
   if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
     curproc->tf->eax = syscalls[num]();
+    #ifdef PRINT_SYSCALLS
+    cprintf("%s -> %d \n", syscallnames[num], num);
+    #endif
   } else {
     cprintf("%d %s: unknown sys call %d\n",
             curproc->pid, curproc->name, num);
diff --git a/syscall.h b/syscall.h
index 7fc8ce1..14f3e17 100644
--- a/syscall.h
+++ b/syscall.h
@@ -22,3 +22,4 @@
 #define SYS_close   SYS_mkdir+1
 #define SYS_halt    SYS_close+1
 // student system calls begin here. Follow the existing pattern.
+#define SYS_date    SYS_halt+1
diff --git a/sysproc.c b/sysproc.c
index 98563ea..8447d97 100644
--- a/sysproc.c
+++ b/sysproc.c
@@ -97,3 +97,17 @@ sys_halt(void)
   return 0;
 }
 #endif // PDX_XV6
+int
+sys_date(void)
+{
+  struct rtcdate *d;
+
+  if(argptr(0, (void*)&d, sizeof(struct rtcdate)) < 0)
+  {
+    return -1;
+  }
+
+  cmostime(d);
+
+  return 0;
+}
diff --git a/user.h b/user.h
index 31d9134..16d054b 100644
--- a/user.h
+++ b/user.h
@@ -43,3 +43,6 @@ int atoi(const char*);
 int atoo(const char*);
 int strncmp(const char*, const char*, uint);
 #endif // PDX_XV6
+#ifdef CS333_P1
+int date(struct rtcdate*);
+#endif // CS333_P1
diff --git a/usys.S b/usys.S
index 0d4eaed..84bd80b 100644
--- a/usys.S
+++ b/usys.S
@@ -30,3 +30,4 @@ SYSCALL(sbrk)
 SYSCALL(sleep)
 SYSCALL(uptime)
 SYSCALL(halt)
+SYSCALL(date)