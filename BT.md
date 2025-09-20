# Vũ Quốc Nghĩa - B23DCAT212
#  Câu 1: Viết mô-đun kernel C cơ bản trên Linux

Mô-đun kernel là đoạn mã có thể **nạp vào kernel (insmod)** hoặc **gỡ ra khỏi kernel (rmmod)** mà không cần phải biên dịch lại toàn bộ kernel.

### a) Code mô-đun kernel (hello.c)

```c
#include <linux/module.h>   // cần cho module_init, module_exit
#include <linux/kernel.h>   // cần cho printk
#include <linux/init.h>     // cần cho macro __init và __exit

// Hàm chạy khi module được nạp vào kernel
static int __init hello_init(void) {
    printk(KERN_INFO "Hello! Module da duoc nap vao kernel.\n");
    return 0; // 0 = thành công
}

// Hàm chạy khi module được gỡ ra khỏi kernel
static void __exit hello_exit(void) {
    printk(KERN_INFO "Bye! Module da duoc go khoi kernel.\n");
}

// Đăng ký hàm init và exit
module_init(hello_init);
module_exit(hello_exit);

// Thông tin module
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Sinh vien PTIT");
MODULE_DESCRIPTION("Vi du mo-dun kernel don gian");
```

### b) Makefile để biên dịch

Tạo file **Makefile**:

```makefile
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

### c) Các lệnh CLI thao tác với mô-đun

```bash
# Biên dịch mô-đun
make

# Nạp mô-đun vào kernel
sudo insmod hello.ko

# Kiểm tra mô-đun đã nạp chưa
lsmod | grep hello

# Xem log tin nhắn từ kernel
dmesg | tail

# Gỡ mô-đun khỏi kernel
sudo rmmod hello

# Kiểm tra lại
lsmod | grep hello
dmesg | tail
```

---

#  Câu 2: Sơ đồ 7 trạng thái của tiến trình

Trong Linux, tiến trình có thể trải qua **7 trạng thái chính**:

1. **New (Mới tạo)** → Tiến trình vừa được tạo (fork).
2. **Ready (Sẵn sàng)** → Đang chờ CPU để chạy.
3. **Running (Đang chạy)** → Được CPU cấp quyền thực thi.
4. **Waiting / Blocked (Chờ)** → Đang chờ I/O hoặc tài nguyên khác.
5. **Terminated (Kết thúc)** → Tiến trình đã hoàn tất.
6. **Suspended / Stopped (Tạm dừng)** → Tiến trình bị dừng (do tín hiệu `SIGSTOP`).
7. **Zombie** → Tiến trình đã kết thúc nhưng chưa được cha gọi `wait()`, nên vẫn còn entry trong bảng tiến trình.

### Chuyển trạng thái:

* New → Ready (khi tiến trình được đưa vào hàng đợi).
* Ready → Running (CPU cấp quyền).
* Running → Waiting (nếu cần I/O).
* Running → Ready (bị CPU thu hồi do hết quantum).
* Running → Terminated (hoàn thành).
* Terminated → Zombie (nếu cha chưa xử lý).
* Waiting → Ready (I/O xong).

---

#  Câu 3: Code C trên Linux sử dụng `getpid()`, `getppid()`, `fork()`, `exit()`

### Code ví dụ:

```c
#include <stdio.h>
#include <unistd.h>   // fork, getpid, getppid
#include <stdlib.h>   // exit

int main() {
    pid_t pid;

    printf("Tien trinh cha: PID = %d, PPID = %d\n", getpid(), getppid());

    pid = fork(); // tạo tiến trình con

    if (pid < 0) {
        perror("fork failed");
        exit(1);
    }
    else if (pid == 0) {
        // Code cho tiến trình con
        printf("Tien trinh con: PID = %d, PPID = %d\n", getpid(), getppid());
        exit(0); // tiến trình con kết thúc
    }
    else {
        // Code cho tiến trình cha
        printf("Tien trinh cha tao con voi PID = %d\n", pid);
        sleep(1); // chờ để thấy con chạy xong
    }

    return 0;
}
```

### Cách chạy:

```bash
gcc fork_demo.c -o fork_demo
./fork_demo
```

### Kết quả (ví dụ):

```
Tien trinh cha: PID = 1234, PPID = 567
Tien trinh cha tao con voi PID = 1235
Tien trinh con: PID = 1235, PPID = 1234
```

---


# Câu 4:
#  Lập trình đa luồng với pthread trong C

## 1. Thư viện pthread

Trong C, thư viện **`<pthread.h>`** cung cấp API để làm việc với đa luồng:

* `pthread_create()` → Tạo một thread mới.
* `pthread_exit()` → Kết thúc thread hiện tại.
* `pthread_join()` → Gộp (chờ) một thread khác kết thúc.

🔹 **Chú ý**:

* Biến **local** (cục bộ) được lưu riêng trên stack từng thread.
* Biến **global** (toàn cục) được chia sẻ giữa tất cả các thread.

---

## 2. Code ví dụ chương trình đa luồng

 File **`thread_demo.c`**:

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

// Biến global (chia sẻ giữa các thread)
int global_counter = 0;

// Hàm mà thread sẽ chạy
void *thread_func(void *arg) {
    int id = *((int *)arg); // lấy id của thread
    int local_counter = 0;  // biến local (riêng từng thread)

    printf("Thread %d bat dau (PID = %ld)\n", id, pthread_self());

    for (int i = 0; i < 3; i++) {
        local_counter++;
        global_counter++;
        printf("Thread %d: local = %d, global = %d\n",
               id, local_counter, global_counter);
        sleep(1);
    }

    printf("Thread %d ket thuc\n", id);
    pthread_exit(NULL);
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        printf("Su dung: %s <so_luong_thread>\n", argv[0]);
        exit(1);
    }

    int n = atoi(argv[1]); // số lượng thread từ tham số dòng lệnh
    pthread_t tid[n];
    int id[n];

    printf("Main: tao %d thread\n", n);

    // Tạo các thread
    for (int i = 0; i < n; i++) {
        id[i] = i + 1;
        pthread_create(&tid[i], NULL, thread_func, &id[i]);
    }

    // Main join các thread (đợi tất cả hoàn thành)
    for (int i = 0; i < n; i++) {
        pthread_join(tid[i], NULL);
    }

    printf("Main ket thuc. Global counter = %d\n", global_counter);
    return 0;
}
```

---

## 3. Cách biên dịch & chạy

```bash
gcc thread_demo.c -o thread_demo -lpthread
./thread_demo 3
```

Ví dụ output:

```
Main: tao 3 thread
Thread 1 bat dau (PID = 139822158968576)
Thread 2 bat dau (PID = 139822150575872)
Thread 3 bat dau (PID = 139822142183168)
Thread 1: local = 1, global = 1
Thread 2: local = 1, global = 2
Thread 3: local = 1, global = 3
...
Thread 3 ket thuc
Thread 2 ket thuc
Thread 1 ket thuc
Main ket thuc. Global counter = 9
```

---

## 4. Trả lời câu hỏi

1. **Chương trình có bao nhiêu thread?**

   * Tổng số luồng = `n` (tham số dòng lệnh) + 1 main thread.
   * Ví dụ: `./thread_demo 3` → 4 luồng (3 con + 1 main).

2. **Thread main có gộp lại với các thread con không?**

   * Có, vì `pthread_join()` được gọi. Nếu không, main có thể kết thúc trước khi thread con chạy xong.

3. **Các luồng có kết thúc theo đúng thứ tự tạo không?**

   * Không chắc. Thứ tự phụ thuộc vào lịch trình CPU → có thể khác nhau mỗi lần chạy.

4. **Nếu chạy nhiều lần, kết quả có thay đổi không?**

   * Có. Thứ tự in ra và giá trị biến `global` sẽ thay đổi do race condition.

---

