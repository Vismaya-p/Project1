extern "C" void kernel_main() {
    const char* msg = "Hello from 64-bit kernel!";
    char* video = (char*)0xB8000;
    for (int i = 0; msg[i]; ++i) {
        video[i * 2] = msg[i];
        video[i * 2 + 1] = 0x07; // White on black
    }

    while (1) {
        __asm__ __volatile__("hlt");
    }
}
