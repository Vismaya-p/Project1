all: os.img

bootloader.bin: bootloader.asm
	nasm -f bin bootloader.asm -o bootloader.bin

kernel.o: kernel.cpp
	x86_64-elf-g++ -ffreestanding -m64 -c kernel.cpp -o kernel.o

kernel.bin: kernel.o linker.ld
	x86_64-elf-ld -T linker.ld -o kernel.bin kernel.o

os.img: bootloader.bin kernel.bin
	dd if=/dev/zero of=os.img bs=512 count=2880
	dd if=bootloader.bin of=os.img bs=512 count=1 conv=notrunc
	dd if=kernel.bin of=os.img bs=512 seek=1 conv=notrunc

run: os.img
	qemu-system-x86_64 -drive format=raw,file=os.img

clean:
	rm -f *.bin *.o os.img
