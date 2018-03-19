# Pertemuan 2 - Docker Swarm

Jika Docker sudah terdiri dari beberapa node, supaya bisa bekerja dengan baik, maka dilakukan *orkestrasi*, ada beberapa cara  untuk *orkestrasi*, salah satunya adalah **Docker Swarm mode**.

## Fungsi Orkestrasi

*Orkestrasi* digunakan agar service yang dijalankan Docker bisa selaras.
Dalam *orkestrasi* dipimpin oleh node, yaitu **manager** dan yang lainnya sebagai **worker**.

**Manager** dan **worker** dapat ditempatkan pada 1 komputer secara fisik, atau bisa lebih dari 1 komputer.
Di sini ada istilah *scale up* dan *scale down*.
*Scale up* berarti menambah komputer atau hanya nodenya, kebalikannya dengan *scale down*.

## Setting Docker Swarm Mode

Di sini kita bisa mencoba Swarm Mode melalui [Katacoda](https://katacoda.com/courses/docker-orchestration/).
Langkahnya adalah :

#### 1.Inisialisasi Swarm Mode**
Cara inisialisasi Docker Swarm mode adalah melalui perintah *init*.

```docker swarm init```

Node pertama yang diinisialisasikan ini akan otomatis menjadi **manager**. Sedangkan selanjutnya bisa kita setting menjadi **manager** atau **worker**, biasanya kita membutuhkan 3 s/d 5 **manager** di production untuk memeastikan ketersediaan service kita.
Dari inisialisasi ini akan dihasilkan sebuah **token** untuk menmabahkan node lain.
Simpan **token*** ini untuk *scaling cluster* kita di kemudian hari jika kite membuahtuhkan *scaling*.

#### 2.Join Cluster

Setelah Swarm mode di *enable* , kita bisa menambahkan node. Jika suatu ketika node *down* misalkan karena *crash*, kontainer yang berjalan di host tersebut akan otomatis mengalihkan ke node lain yang aktif. Ini menjanjikan *high-availability* untuk service kita.

Kita dapat bergabung dengan manager dari sebuah cluster ( dalam hal ini yang telah kita buat sebelumnya ) dengan menggunakan token yang telah di generat sebelumnya.

Docker menggunakan port 2377 untuk Swarm Mode, yang biasanya di blok dari public dan hanya bisa diakses oleh *trusted user*, di sini direkomendasikan menggunakan VPN atau private network.

Cara join cluster adalah :
  - Pada node yang akan join, kita jalankan :
    
    ```token=$(docker -H 172.17.0.19:2345 swarm join-token -q worker) && echo $token```
    
    Di sini node yang akan join akan bertanya token ke manager ( node dengan ip 172.17.0.19 ) melalui *swarm join-token* yang
    mana node ini akan join sebagai worker dan menjadi nilai dari paramtere **token**.
    
    Kemudian kita jalankan :
    
    ```docker swarm join 172.17.0.19:2377 --token $token```
    
    Di sini node akan join ke manager menggunakan paramter **$token** yang telah digenerate sebelumnya.
    
   - Pada node manger , kita jalankan :
   
   ```docker node ls``` 
   
   maka akan terlihat node sekarang menjadi 2, **ID** dengan tanda * merupakan **Leader** atau **manager**.
   
#### 3. Buat Overlay Network

*Overlay network* digunakan untuk meng-*enable* kontainer yang ada di host yang berbeda untuk berkomunikasi. Sangat mendukung untuk *deployment* skala besar pada *cloud*.
Di sini kita akan membuat *overlay network* yang disebut *skynet*. Semua container yang terdaftar pada jaringan ini dapat berkomunikasi.

Caranya kita jalankan perintah berikut pada **MANAGER**:

```docker network create -d overlay skynet```

#### 4.Deploy Service

Pada **MANAGER** kita jalankan :

```docker service create --name http --network skynet --replicas 2 -p 80:80 katacoda/docker-http-server```

Dari perintah di atas kita akan men-*deploy* Docker Image dengan nama *Katacoda/docker-http-server* dimana htpp akan di *attach* pada *skynet*. Untuk memastikan replikasi dan *availability* di sini kita buat 2 *instance*. Kemudian kita lakukan *load balance* 2 kontainer ini pada port 80.

Untuk melihat service pada **MANAGER**, kita gunakan perintah :

```docker service ls```

Untuk melihat list container pada node pertama (manager) atau kedua (worker), bisa kita jalankan :

```docker ps```

Jika kita ingin menjalankan HTTP pada port publik kita, ini akan dijalankan oleh 2 kontainer, lalu kita jalankan :

```curl docker```

#### 5.Memeriksa Service

Kita dapat memeriksa service kita running atau tidak pada masing-masing kontainer dengan perintah :

```docker service ps http```

Kemudian kita dapat melihat detail konfigurasi dengan perintah :

```docker service inspect --pretty http```

Pada masing-masing node kita bisa melihat task apa yang sedang run dengan perintah :

```docker node ps self```

*self* ini mengacu pada manager

Kita juga bisa menggunakan ID dari sebuah node :

```docker node ps $(docker node ls -q | head -n1)```

> Perintah pada poin 5 ini dijalankan di manager bukan di worker

#### 6.Scaling Service

Service bisa kita scale menjadi beberapa instance yang run di cluster.
Kita dapat men-*scale* service http yang sebelumnya 2 menjadi 5 ( dalam hal ini *scalling up* ) dengan perintah :

```docker service scale http=5```

Load balancer otomatis terupdate dan request ke service http ini sekarang diproses oleh container, kemudian kita jalanan :

```curl docker```

Untuk *scalling down* sama intinya,kita tinggal mengganti nilai angka 5 menjadi nilai yang lebih kecil

