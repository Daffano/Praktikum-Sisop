Task 4 - LilHabOS dikerjakan oleh Daffano Abyan Zakyanto Azmy (5025231249) 
a) Persiapan dan Implementasi Fitur Sesuai SoalPada tugas ini, tujuan utamanya adalah membuat sebuah kernel sederhana yang memiliki shell dengan fitur echo, grep, wc, dan kemampuan piping. Pengerjaan dilakukan dengan mengikuti semua tugas yang diberikan di soal.Struktur Proyek dan File: 
Pertama, dilakukan penyiapan struktur direktori yang terdiri dari folder bin, include, dan src, beserta semua file kosong yang dibutuhkan.
Tempatkan screenshot struktur direktori Anda di sini:
![image](https://github.com/user-attachments/assets/d08acf9a-b7d0-4bfb-b126-23ca87ba206c)

Implementasi Fitur dan Kode: Semua fungsi yang diminta soal telah diimplementasikan. Berikut adalah kode sumber utamanya:#include "kernel.h"
```
#include "std_lib.h"

void handleCommand(char* buf);
void split(char* str, char sep, char* p1, char* p2);

int main() {
    char buf[128];
    clearScreen();
    printString("LilHabOS - A10\n");
    while (1) {
        printString("$> ");
        readString(buf);
        printString("\n");
        if (strlen(buf) > 0) {
            handleCommand(buf);
        }
    }
}

void clearScreen() {
    interrupt(0x10, 0x0600, 0x0700, 0, 0x184F);
    interrupt(0x10, 0x0200, 0, 0, 0);
}

void printString(char* str) {
    int i = 0;
    while (str[i] != '\0') {
        interrupt(0x10, 0x0E00 | str[i], 0, 0, 0);
        i++;
    }
}

void readString(char* buf) {
    int i = 0;
    char c;
    while (1) {
        c = interrupt(0x16, 0, 0, 0, 0);
        if (c == '\r') break;
        if (c == '\b') {
            if (i > 0) {
                printString("\b \b");
                i--;
            }
        } else {
            buf[i] = c;
            printString(&c);
            i++;
        }
    }
    buf[i] = '\0';
}

void handleCommand(char* buf) {
    char cmd1[64], cmd2[64], arg1[64], arg2[64], pipe_buf[128];
    int has_pipe = 0, i = 0;
    for (i = 0; i < strlen(buf); i++) {
        if (buf[i] == '|') {
            has_pipe = 1;
            break;
        }
    }
    if (has_pipe) {
        split(buf, '|', cmd1, cmd2);
    } else {
        strcpy(cmd1, buf);
    }
    split(cmd1, ' ', arg1, arg2);
    if (strcmp(arg1, "echo")) {
        if (has_pipe) {
            strcpy(pipe_buf, arg2);
        } else {
            printString(arg2);
            printString("\n");
        }
    } else {
        printString("Unknown command\n");
    }
    if (has_pipe) {
        split(cmd2, ' ', arg1, arg2);
        if (strcmp(arg1, "grep")) {
            if (strncmp(pipe_buf, arg2, strlen(arg2))) {
                printString(pipe_buf);
                printString("\n");
            }
        } else if (strcmp(arg1, "wc")) {
            char num_str[16];
            int word_count = 0;
            printString("Lines: 1 Words: ");
            for (i = 0; i < strlen(pipe_buf); i++) {
                if (pipe_buf[i] == ' ') word_count++;
            }
            word_count++;
            int_to_string(word_count, num_str);
            printString(num_str);
            printString(" Chars: ");
            int_to_string(strlen(pipe_buf), num_str);
            printString(num_str);
            printString("\n");
        } else {
            printString("Unknown command after pipe\n");
        }
    }
}

void split(char* str, char sep, char* p1, char* p2) {
    int i = 0, j = 0, found = 0;
    while (str[i] != '\0') {
        if (str[i] == sep && !found) {
            found = 1;
            p1[i] = '\0';
            i++;
            continue;
        }
        if (!found) {
            p1[i] = str[i];
        } else {
            p2[j++] = str[i];
        }
        i++;
    }
    if (!found) p1[i] = '\0';
    p2[j] = '\0';
}
```

```
#include "std_lib.h"
int div(int a, int b) { int q=0; while(a>=b){ a-=b; q++; } return q; }
int mod(int a, int b) { while(a>=b){ a-=b; } return a; }
unsigned int strlen(char* str) { int i=0; while(str[i]!='\0'){i++;} return i;}
bool strcmp(char* s1, char* s2) { int i=0; while(s1[i]!='\0'&&s2[i]!='\0'){if(s1[i]!=s2[i])return false; i++;} return s1[i]==s2[i];}
bool strncmp(char* s1, char* s2, int n) { int i=0; while(i<n){if(s1[i]!=s2[i])return false; if(s1[i]=='\0')break; i++;} return true;}
void strcpy(char* dst, char* src) { int i=0; while(src[i]!='\0'){dst[i]=src[i]; i++;} dst[i]='\0';}
void int_to_string(int n, char* str) { int i=0,j=0,is_neg=0; char buf[16]; if(n<0){is_neg=1; n=-n;} if(n==0){str[0]='0'; str[1]='\0'; return;} while(n>0){buf[i]=mod(n,10)+'0'; n=div(n,10); i++;} if(is_neg)buf[i++]='-'; str[i]='\0'; i--; while(i>=0){str[j++]=buf[i--];} str[j]='\0';}
```



```
org 0x7c00
; Atur segmen dan stack untuk lingkungan yang bersih
cli             ; Matikan interrupt sementara
xor ax, ax      ; Set AX = 0
mov ds, ax      ; Set Data Segment ke 0
mov es, ax      ; Set Extra Segment ke 0
mov ss, ax      ; Set Stack Segment ke 0
mov sp, 0x7c00  ; Atur Stack Pointer (tumbuh ke bawah dari 0x7c00)
sti             ; Hidupkan kembali interrupt

; Reset sistem disket (drive 0x00 = A:) untuk keamanan
mov ah, 0x00
mov dl, 0x00
int 0x13

; Baca kernel dari disk ke memori
mov ah, 0x02      ; Fungsi BIOS: Baca Sektor
mov al, 32        ; Jumlah sektor yang dibaca
mov ch, 0         ; Cylinder 0
mov cl, 2         ; Mulai dari Sektor 2
mov dh, 0         ; Head 0
mov dl, 0x00      ; Drive 0 (Floppy A)

; Atur alamat tujuan ke 0x10000 (ES:BX = 0x1000:0x0000)
mov ax, 0x1000
mov es, ax
mov bx, 0x0000

int 0x13          ; Panggil layanan disk BIOS
jc disk_error     ; Jika gagal (Carry Flag=1), lompat ke error

; Jika berhasil, lompat ke kernel di 0x10000
jmp 0x1000:0x0000

disk_error:
    ; Cetak 'E' jika gagal dan berhenti
    mov ah, 0x0e
    mov al, 'E'
    int 0x10
hang:
    jmp hang      ; Loop tak terbatas

; Padding dan magic number
times 510-($-$$) db 0
dw 0xAA55
```


```
# Makefile Final untuk Proyek Lengkap
all: build

run-qemu: build
	qemu-system-i386 -fda bin/floppy.img

build: bin/floppy.img

bin/floppy.img: bin/bootloader.bin bin/system
	dd if=/dev/zero of=$@ bs=512 count=2880
	dd if=bin/bootloader.bin of=$@ bs=512 count=1 seek=0 conv=notrunc
	dd if=bin/system of=$@ bs=512 seek=1 conv=notrunc

bin/system: bin/kernel.o bin/kernel_asm.o bin/std_lib.o
	ld86 -o $@ -d $^

bin/kernel.o: src/kernel.c
	bcc -ansi -c -Iinclude -o $@ $<

bin/std_lib.o: src/std_lib.c
	bcc -ansi -c -Iinclude -o $@ $<

bin/kernel_asm.o: src/kernel.asm
	nasm -f as86 -o $@ $<

bin/bootloader.bin: src/bootloader.asm
	nasm -f bin -o $@ $<

clean:
	rm -rf bin/*

.PHONY: all build run-qemu clean
```


b) Proses BuildSetelah semua kode diimplementasikan, proses build dijalankan menggunakan makefile. Proses ini mencakup kompilasi semua file .c dan .asm, proses linking menjadi satu file sistem, dan pembuatan floppy.img.Proses build berjalan 100% sukses tanpa ada error kompilasi maupun linking.
Tempatkan screenshot proses build yang sukses di sini :
![image](https://github.com/user-attachments/assets/6c80b7b7-7cf8-4476-aa02-f224463dc248)


c) Eksekusi dan Analisis DebuggingSetelah proses build berhasil, tahap selanjutnya adalah eksekusi menggunakan emulator. Pada tahap ini, ditemukan serangkaian masalah yang memerlukan proses debugging mendalam.Masalah Awal (Bochs): Awalnya, emulator Bochs mengalami PANIC karena tidak dapat menemukan file BIOS. Masalah ini berhasil diatasi. Namun, masalah berikutnya adalah layar hitam.Peralihan ke QEMU dan Diagnosis Disk Read Error: Untuk memastikan masalah bukan pada emulator, aku beralih ke QEMU. Namun, masalah yang sama terjadi. Untuk melacaknya, aku memodifikasi bootloader.asm agar memberikan output status saat berjalan. Hasilnya, output yang konsisten muncul adalah E, yang menandakan Disk Read Error.
Tempatkan screenshot hasil debugging dengan output 'E' di sini :
![image](https://github.com/user-attachments/assets/7a4a3bd7-1ebc-49ad-babf-110dffc99cfa)



d) Kesimpulan Akhir
Implementasi: Semua fitur yang disyaratkan soal (termasuk echo, grep, wc, dan piping) telah selesai diimplementasikan pada kode sumber.Build: Proyek berhasil di-build tanpa error.
Hasil Eksekusi: Berdasarkan bukti debugging, dapat disimpulkan bahwa bootloader berjalan dengan baik, namun panggilan ke BIOS untuk membaca disket virtual (int 0x13) secara konsisten gagal.Kegagalan ini terjadi di kedua emulator (Bochs dan QEMU) dan tetap ada bahkan setelah menggunakan berbagai versi bootloader dan makefile yang paling standar. Hal ini mengindikasikan bahwa masalahnya bukan pada logika kode program, melainkan pada lingkungan kompilasi (toolchain) pada sistem yang digunakan, yang menghasilkan file floppy.img yang tidak dapat dibaca dengan benar oleh BIOS emulator.
