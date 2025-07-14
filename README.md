; bootloader.asm - Bootloader that switches to 64-bit mode
; Assembled with: nasm -f bin bootloader.asm -o bootloader.bin

BITS 16
ORG 0x7C00

start:
    cli
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00

    ; Load kernel from disk into 0x100000
    mov ah, 0x02         ; Read sectors
    mov al, 20           ; Number of sectors
    mov ch, 0
    mov cl, 2
    mov dh, 0
    mov dl, 0x80         ; HDD
    mov bx, 0x0000
    mov es, 0x1000       ; 0x100000 = 0x1000:0x0000
    int 0x13
    jc load_error

    ; GDT setup
    lgdt [gdt_descriptor]

    ; Enter Protected Mode
    mov eax, cr0
    or eax, 1
    mov cr0, eax

    jmp CODE32_SEG:init_pm32

[BITS 32]
init_pm32:
    ; Set up data segments
    mov ax, DATA32_SEG
    mov ds, ax
    mov ss, ax
    mov es, ax
    mov fs, ax
    mov gs, ax

    ; Enable PAE
    mov ecx, cr4
    or ecx, 1 << 5
    mov cr4, ecx

    ; Setup page tables (identity map first 2MB)
    mov eax, page_table_l4
    mov cr3, eax

    ; Enable Long Mode
    mov ecx, 0xC0000080       ; IA32_EFER MSR
    rdmsr
    or eax, 1 << 8            ; LME
    wrmsr

    ; Enable paging
    mov eax, cr0
    or eax, 0x80000000        ; Set PG bit
    mov cr0, eax

    ; Far jump to 64-bit kernel
    jmp CODE64_SEL:long_mode_start

[BITS 64]
long_mode_start:
    ; Now we're in 64-bit long mode
    mov ax, DATA64_SEL
    mov ds, ax
    mov ss, ax

    ; Jump to 64-bit kernel entry
    mov rsi, kernel_entry
    call rsi

hang:
    hlt
    jmp hang

load_error:
    mov si, error_msg
    call print_string
    jmp $

print_string:
    mov ah, 0x0E
.print_char:
    lodsb
    cmp al, 0
    je .done
    int 0x10
    jmp .print_char
.done:
    ret

error_msg db "Disk error", 0

; ---------------------------
; GDT Setup
align 8
gdt:
    dq 0x0000000000000000        ; Null
    dq 0x00AF9A000000FFFF        ; Code32
    dq 0x00AF92000000FFFF        ; Data32
    dq 0x00AF9A000000FFFF        ; Code64
    dq 0x00AF92000000FFFF        ; Data64

gdt_descriptor:
    dw gdt_end - gdt - 1
    dd gdt

gdt_end:

CODE32_SEG equ 0x08
DATA32_SEG equ 0x10
CODE64_SEL equ 0x18
DATA64_SEL equ 0x20

; ---------------------------
; Page Tables
align 4096
page_table_l4:
    dq page_table_pdpt | 0x03
align 4096
page_table_pdpt:
    dq page_table_pd | 0x03
align 4096
page_table_pd:
    dq 0x0000000000000083     ; 2MB identity map, Present | Write | 2MB

; ---------------------------
; Kernel jump location
kernel_entry dq 0x100000

; Boot signature
times 510 - ($ - $$) db 0
dw 0xAA55
