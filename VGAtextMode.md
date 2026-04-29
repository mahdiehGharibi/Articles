# How I Printed My First Character on Screen in a 32-bit Kernel
by Tinar - QtinarOS

## 1. The Problem: A Silent Kernel

I was following the OSDev "Bare Bones" tutorial. I built a cross-compiler, wrote a linker script, and finally saw GRUB load my kernel. But after that? Nothing. A completely black screen. I didn't know if my kernel had crashed, if the bootloader failed, or if everything was fine but I just couldn't see it.

I needed a way to prove that my kernel was actually running. The simplest solution? Print a message on the screen.

## 2. The Solution: VGA Text Mode

The standard way to output text on x86 PCs (without a full graphics driver) is VGA text mode. It's a special memory area that the VGA hardware continuously reads and displays.

1. Memory address: `0x88000` (physical)
2. Resolution: 25 rows, 80 columns (by default)
3. Each character cell: 2 bytes  
   - Byte 0: ASCII code of the character  
   - Byte 1: Color attribute (foreground and background)

The attribute byte is split into two nibbles:

- Lower 4 bits: foreground color (e.g., `0x0` black, `0xF` white)
- Upper 4 bits: background color

For example, `0x07` means light grey (7) on black (0).

## 3. The Code - Step by Step

I wrote two small functions in C (freestanding environment, no standard library).

### 3.1 Writing a single character

```c
#define VGA_ADDR  0x88000
#define VGA_WIDTH 80
#define VGA_HEIGHT 25

void putchar_at(char ch, int row, int col, char color) {
    char *video = (char*) VGA_ADDR;
    int offset = (row * VGA_WIDTH + col) * 2;
    video[offset] = ch;
    video[offset + 1] = color;
}
```
### 3.2 Printing a string
I wanted to print strings easily, so I made a simple kprint that starts from the top-left corner and moves forward.
```c
static int cursor_row = 0;
static int cursor_col = 0;
static char current_color = 0x07; // light grey on black

void kprint(const char *str) {
    char *video = (char*) VGA_ADDR;
    while (*str) {
        if (*str == '\n') {
            cursor_row++;
            cursor_col = 0;
        } else {
            int offset = (cursor_row * VGA_WIDTH + cursor_col) * 2;
            video[offset] = *str;
            video[offset + 1] = current_color;
            cursor_col++;
            if (cursor_col >= VGA_WIDTH) {
                cursor_col = 0;
                cursor_row++;
            }
        }
        str++;
    }
}
```

### 3.3 The kernel main function
```c
void kernel_main() {
    kprint("Hello, Qtinar OS world!\n");
    kprint("This is my first printed message.\n");
    while (1); // hang forever
}
```
## 4. What Happened When I Ran It
I compiled the kernel with my cross-compiler:
```bash
i686-elf-gcc -c kernel.c -o kernel.o -std=gnu99 -ffreestanding -O2 -Wall -Wextra
i686-elf-ld -T linker.ld -o myos.bin boot.o kernel.o
```
Then I ran it in `QEMU`:
```bash
qemu-system-i386 -kernel myos.bin
```
And I saw this on the screen:
```text
Hello, Qtinar OS world!
This is my first printed message.
```
## 5. What I Learned
- Direct memory access is real – you can write to `0x88000` and see changes immediately.

- No driver is needed for basic text output – the VGA hardware does all the work.

- A simple kprint function is enough to debug almost everything later (register dumps, panic messages, etc.).

## 6. Next Steps
Now that I can print text, I will:

- Add support for printing numbers (very useful for debugging).

- Move the cursor automatically when the screen fills up.

- Set up interrupts (keyboard and timer) to make the OS interactive.

After that, I will start implementing a simple round-robin scheduler – the first step toward a real multitasking OS.

`Thank you for reading. I am a 20-year-old developer from Iran, and this is my journey of building the Qtinar OS.`
