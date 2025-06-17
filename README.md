# BACKEND 
1. env nya = sesuaikan ke dengan nama database

```
 database.default.hostname = 127.0.0.1
 database.default.database = cuti
 database.default.username = root
 database.default.password =
 database.default.DBDriver = MySQLi
# database.default.DBPrefix =
 database.default.port = 3306
```

2. Cofig > Routes.php 
```
<?php

use CodeIgniter\Router\RouteCollection;

/**
 * @var RouteCollection $routes
 */

// Home default
$routes->get("/", "Home::index");

$routes->group('dosen', function($routes) {
    $routes->get('/', 'Dosen::index'); // GET /dosen
    $routes->get('nama/(:segment)', 'Dosen::showName/$1'); // GET /dosen/nama/{nama}
    $routes->get('(:segment)', 'Dosen::show/$1'); // GET /dosen/{id}
    $routes->post('/', 'Dosen::create'); // POST /dosen
    $routes->put('(:segment)', 'Dosen::update/$1'); // PUT /dosen/{id}
    $routes->delete('(:segment)', 'Dosen::delete/$1'); // DELETE /dosen/{id}
});

// Mahasiswa
$routes->get("mahasiswa/showName/(:any)", 'Mahasiswa::showName/$1');
$routes->get("mahasiswa/(:segment)", 'Mahasiswa::show/$1');
$routes->put("mahasiswa/(:segment)", "Mahasiswa::update/$1");
$routes->put("user/(:segment)", "User::update/$1");
$routes->get("mahasiswa", "Mahasiswa::index");
$routes->delete("mahasiswa/(:any)", 'Mahasiswa::delete/$1');
$routes->delete("user/(:any)", 'User::delete/$1');
$routes->post("mahasiswa", "Mahasiswa::create"); // <-- ini penting
$routes->post("user", "User::create");
$routes->put("mahasiswa/(:segment)", "Mahasiswa::update/$1"); // biar PUT juga aman

// Cuti
$routes->get("cuti/npm/(:any)", 'Cuti::getCutiByNpm/$1');
$routes->post("cuti", "Cuti::create");
$routes->post("cuti/(:segment)", "Cuti::createWithNpm/$1"); // Kalau kamu butuh bisa dipakai
$routes->resource("cuti");
$routes->post("pengajuancuti", "PengajuanCuti::getMahasiswaCuti");
$routes->get("pengajuan-cuti", "PengajuanCuti::index");

// User
$routes->get("user/showName/(:any)", 'User::showName/$1');
$routes->get("/user", "User::index");
$routes->post("/riwayatCuti", "RiwayatMhs::getCuti");
$routes->post("/mhsberanda", "MhsBeranda::getMahasiswa");
$routes->get("/pengajuancuti", "PengajuanCuti::getMahasiswaCuti");
$routes->post("/riwayatadmin", "RiwayatAdmin::getRiwayatAdmin");
$routes->post("/viewberandamahasiswa", "BerandaMhs::getBerandaMahasiswa");
$routes->post("/viewriwayatadmin", "RiwayatAdmView::getRiwayatAdmin");
$routes->post("/viewriwayatmahasiswa", "RiwayatMhsView::getMahasiswaCuti");

// Kajur
$routes->options("kajur/(:segment)", 'Kajur::options/$1');
$routes->get("kajur/(:segment)", 'Kajur::show/$1');
$routes->resource("kajur");
$routes->post("/viewberandakajur", "KajurBrnd::getBerandaKajurCntrl");

// Admin
$routes->resource("admin");

// Global CORS preflight (OPTIONS) handling
$routes->options("(:any)", "Home::options");
```

3. Controller 
- Dosen.php
```
<?php

namespace App\Controllers;

use App\Models\UserModel;
use App\Models\DosenModel;
use App\Models\KajurModel;
use App\Models\MahasiswaModel;
use CodeIgniter\API\ResponseTrait;
use CodeIgniter\HTTP\ResponseInterface;

// Menambahkan Header CORS Global
header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS");
header(
    "Access-Control-Allow-Headers: Content-Type, X-Requested-With, Authorization"
);

class Dosen extends BaseController
{
    use ResponseTrait;
    protected $userModel;
    protected $mahasiswaModel;
    protected $dosenModel;
    protected $kajurModel;

    public function __construct()
    {
        $this->userModel = new UserModel();
        $this->dosenModel = new DosenModel();
        $this->mahasiswaModel = new MahasiswaModel();
        $this->kajurModel = new KajurModel();
    }

    public function index()
    {
        $data = $this->dosenModel->findAll();
        return $this->respond($data, 200);
    }

    public function show($id = null)
    {
        $data = $this->dosenModel->where("id_dosen", $id)->findAll();
        if ($data) {
            return $this->respond($data, 200);
        } else {
            return $this->failNotFound("Data tidak ditemukan");
        }
    }

    public function showName($nama = null)
    {
        try {
            if (empty($nama)) {
                return $this->fail("Nama mahasiswa harus diisi", 400);
            }

            $data = $this->dosenModel
                ->where("nama_dosen", $nama)
                ->findAll();

            if (!empty($data)) {
                return $this->respond(
                    [
                        "status" => 200,
                        "message" => "Data Dosen ditemukan",
                        "data" => $data,
                    ],
                    200
                );
            } else {
                return $this->failNotFound(
                    "Data mahasiswa dengan nama " . $nama . " tidak ditemukan"
                );
            }
        } catch (\Exception $e) {
            return $this->fail("Terjadi kesalahan: " . $e->getMessage(), 500);
        }
    }

    public function create()
    {
        $data = $this->request->getJSON(true); // ini bagian penting!

        $userCheck = $this->userModel
            ->where("id_user", $data["id_user"])
            ->where("username", $data["nama_dosen"])
            ->first();

        if (!$userCheck) {
            return $this->fail([
                "message" => "ID User tidak sesuai dengan yang ada di tabel user / Nama mahasiswa tidak ada di table user",
            ], 400);
        }

        if (!$this->dosenModel->save($data)) {
            return $this->fail($this->dosenModel->errors());
        }

        return $this->respond([
            "status" => 200,
            "message" => ["success" => "Berhasil Menambah Data"],
        ], 200);
    }

    public function update($id = null)
    {
        $data = $this->request->getJSON(true);
        $data["id_dosen"] = $id;

        $ifExist = $this->dosenModel->where("id_dosen", $id)->findAll();
        if (!$ifExist) {
            return $this->failNotFound("Data tidak ditemukan");
        }

        $npmCheck = $this->dosenModel
            ->where("id_dosen", $data["id_dosen"])
            ->where("id_dosen !=", $id)
            ->first();
        if ($npmCheck) {
            return $this->fail(["message" => "NPM sudah digunakan"], 400);
        }

        $userCheck = $this->userModel
            ->where("id_user", $data["id_user"])
            ->where("username", $data["nama_dosen"])
            ->first();
        if (!$userCheck) {
            return $this->fail(
                [
                    "message" =>
                    "ID User dan username tidak sesuai dengan data di tabel user",
                ],
                400
            );
        }

        if (!$this->dosenModel->save($data)) {
            return $this->fail($this->dosenModel->errors());
        }

        return $this->respond(
            [
                "status" => 200,
                "message" => ["success" => "Berhasil Mengubah Data"],
            ],
            200
        );
    }

    public function delete($id_dosen = null)
    {
        $dosen = $this->dosenModel->where("id_dosen", $id_dosen)->first();
        $id_dosen = $dosen["id_dosen"];
        $dosen = $this->dosenModel->where("id_dosen", $id_dosen)->first();

        if ($dosen) {
            $this->dosenModel->where("id_dosen", $id_dosen)->delete();
            return $this->respondDeleted([
                "status" => 200,
                "message" => ["success" => "Data berhasil dihapus."],
            ]);
        } else {
            return $this->failNotFound("Data dengan NPM $id_dosen tidak ditemukan.");
        }
    }
}
```

- Mahasiswa.php
```
<?php

namespace App\Controllers;

use App\Models\UserModel;
use App\Models\DosenModel;
use App\Models\KajurModel;
use App\Models\MahasiswaModel;
use CodeIgniter\API\ResponseTrait;

// Menambahkan Header CORS Global
header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS");
header(
    "Access-Control-Allow-Headers: Content-Type, X-Requested-With, Authorization"
);

class Mahasiswa extends BaseController
{
    use ResponseTrait;
    protected $userModel;
    protected $mahasiswaModel;
    protected $dosenModel;
    protected $kajurModel;

    public function __construct()
    {
        $this->userModel = new UserModel();
        $this->dosenModel = new DosenModel();
        $this->mahasiswaModel = new MahasiswaModel();
        $this->kajurModel = new KajurModel();
    }

    public function index()
    {
        $data = $this->mahasiswaModel->findAll();
        return $this->respond($data, 200);
    }

    public function show($id = null)
    {
        $data = $this->mahasiswaModel->where("npm", $id)->findAll();
        if ($data) {
            return $this->respond($data, 200);
        } else {
            return $this->failNotFound("Data tidak ditemukan");
        }
    }

    public function showName($nama = null)
    {
        try {
            if (empty($nama)) {
                return $this->fail("Nama mahasiswa harus diisi", 400);
            }

            $data = $this->mahasiswaModel
                ->where("nama_mahasiswa", $nama)
                ->findAll();

            if (!empty($data)) {
                return $this->respond(
                    [
                        "status" => 200,
                        "message" => "Data mahasiswa ditemukan",
                        "data" => $data,
                    ],
                    200
                );
            } else {
                return $this->failNotFound(
                    "Data mahasiswa dengan nama " . $nama . " tidak ditemukan"
                );
            }
        } catch (\Exception $e) {
            return $this->fail("Terjadi kesalahan: " . $e->getMessage(), 500);
        }
    }

    public function create()
    {
        $data = $this->request->getJSON(true);

        $npmCheck = $this->mahasiswaModel->where("npm", $data["npm"])->first();
        if ($npmCheck) {
            return $this->fail(["message" => "NPM sudah digunakan"], 400);
        }

        $userCheck = $this->userModel
            ->where("id_user", $data["id_user"])
            ->where("username", $data["nama_mahasiswa"])
            ->first();
        if (!$userCheck) {
            return $this->fail(
                [
                    "message" =>
                        "ID User tidak sesuai dengan yang ada di tabel user / Nama mahasiswa tidak ada di table user",
                ],
                400
            );
        }

        $kajurCheck = $this->kajurModel
            ->where("id_kajur", $data["id_kajur"])
            ->first();
        if (!$kajurCheck) {
            return $this->fail(
                ["message" => "ID Kajur tidak ditemukan di tabel kajur"],
                400
            );
        }

        if (!$this->mahasiswaModel->save($data)) {
            return $this->fail($this->mahasiswaModel->errors());
        }

        return $this->respond(
            [
                "status" => 200,
                "message" => ["success" => "Berhasil Menambah Data"],
            ],
            200
        );
    }

    public function update($id = null)
    {
        $data = $this->request->getJSON(true);
        $data["npm"] = $id;

        $ifExist = $this->mahasiswaModel->where("npm", $id)->findAll();
        if (!$ifExist) {
            return $this->failNotFound("Data tidak ditemukan");
        }

        $npmCheck = $this->mahasiswaModel
            ->where("npm", $data["npm"])
            ->where("npm !=", $id)
            ->first();
        if ($npmCheck) {
            return $this->fail(["message" => "NPM sudah digunakan"], 400);
        }

        $userCheck = $this->userModel
            ->where("id_user", $data["id_user"])
            ->where("username", $data["nama_mahasiswa"])
            ->first();
        if (!$userCheck) {
            return $this->fail(
                [
                    "message" =>
                        "ID User dan username tidak sesuai dengan data di tabel user",
                ],
                400
            );
        }

        $kajurCheck = $this->kajurModel
            ->where("id_kajur", $data["id_kajur"])
            ->first();
        if (!$kajurCheck) {
            return $this->fail(
                ["message" => "ID Kajur tidak ditemukan di tabel kajur"],
                400
            );
        }

        if (!$this->mahasiswaModel->save($data)) {
            return $this->fail($this->mahasiswaModel->errors());
        }

        return $this->respond(
            [
                "status" => 200,
                "message" => ["success" => "Berhasil Mengubah Data"],
            ],
            200
        );
    }

    public function delete($npm = null)
    {
        $mahasiswa = $this->mahasiswaModel->where("npm", $npm)->first();

        if ($mahasiswa) {
            $this->mahasiswaModel->where("npm", $npm)->delete();
            return $this->respondDeleted([
                "status" => 200,
                "message" => ["success" => "Data berhasil dihapus."],
            ]);
        } else {
            return $this->failNotFound("Data dengan NPM $npm tidak ditemukan.");
        }
    }
}
```

4. Models 
- DosenModel.php
```
<?php

namespace App\Models;

use CodeIgniter\Model;

class DosenModel extends Model
{
    protected $table            = 'dosen_wali';
    protected $primaryKey       = 'id_dosen';
    protected $useAutoIncrement = true;
    protected $returnType       = 'array';
    protected $useSoftDeletes   = false;
    protected $protectFields    = true;
    protected $allowedFields    = [
        "id_dosen", "nama_dosen", "nidn","id_user"
    ];

    protected bool $allowEmptyInserts = false;
    protected bool $updateOnlyChanged = true;

    protected array $casts = [];
    protected array $castHandlers = [];

    // Dates
    protected $useTimestamps = false;
    protected $dateFormat    = 'datetime';
    protected $createdField  = 'created_at';
    protected $updatedField  = 'updated_at';
    protected $deletedField  = 'deleted_at';

    // Validation
    protected $validationRules      = [];
    protected $validationMessages   = [];
    protected $skipValidation       = false;
    protected $cleanValidationRules = true;

    // Callbacks
    protected $allowCallbacks = true;
    protected $beforeInsert   = [];
    protected $afterInsert    = [];
    protected $beforeUpdate   = [];
    protected $afterUpdate    = [];
    protected $beforeFind     = [];
    protected $afterFind      = [];
    protected $beforeDelete   = [];
    protected $afterDelete    = [];
}
```

- MahasiswaModel.php
```
<?php
namespace App\Models;
use CodeIgniter\Model;

class MahasiswaModel extends Model
{
    protected $table = "mahasiswa";
    protected $primaryKey = "npm";
    protected $useAutoIncrement = false;

    protected $allowedFields = [
        "npm",
        "id_user",
        "id_dosen",
        "id_kajur",
        "nama_mahasiswa",
        "tempat_tanggal_lahir",
        "jenis_kelamin",
        "alamat",
        "agama",
        "angkatan",
        "program_studi",
        "no_hp",
        "email",
    ];
}
```

-- jangan lupa composer install dan php spark serve

## Test Postman
1. post http://localhost:8080/dosen/
```
{
    "NIDN": "987654321",
    "nama_dosen": "Dr. Budi Santoso S"
}
```
2. put http://localhost:8080/dosen/230202029
```
{
    "NIDN": "230202029",
    "nama_dosen": "Devia"
}
```
