ada 2 database, database server Utama dan database server client. setiap ajaran baru database client akan di push ke server lalu client kosong lagi. setiap aplikasi akan dibuatkan virtual server (client)

akun siswa dibuat uniq password. ketika pertama login menggunakan password default akan wajib mengganti password. dan ketika lupa password maka akan input data tertentu untuk password di kembalikan ke default. dan login untuk merubahya (tujuannya agar tidak merepotkan petugas,dan tidak membebani whatsapp gateway jika menggunakan OTP). atau mungkin bisa menggunakan OTP lewat Gmail.

diutamakan dasboard piket


setelah saya mencoba aplikasi ini, ternyata masih perlu penyesuaian.

penyesuaian secara fitur :
1. side bar tidak bisa di scroll, dan tidak bisa di hide 
2. menu kampus tidak ada CRUD
3. menu Kelas & Jurusan tidak ada CRUD
4. warna pembeda legend jadwal kurang kontras. buat dengan warna yang berbeda di setiap legend tapi tetap selarah dengan style warna yang sudah ada
5. tambahkan fitur tambah manual untuk guru dan siswa

Temuan Kegagalan sistem :
terkadang menu tidak bisa di klik dan memunculkan error seperti ini :
Unhandled Runtime Error
Error: Cookies can only be modified in a Server Action or Route Handler. Read more: https://nextjs.org/docs/app/api-reference/functions/cookies#cookiessetname-value-options

Penyesuaian desain UI/UX :
1. backgound button keluar warnanya masih sama dengan warna background untama, buat menjadi warna merah agar lebih kontras.
2. masih banyak field yang warnanya sama dengan warna background. buat field dengan warna yang sedikit berbeda agar user mudah mengidentifikasi bahwa itu adalah sebuah field yang dapat diisi.

