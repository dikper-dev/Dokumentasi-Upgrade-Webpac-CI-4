<h1 align="center">Ngeureuyeuh Webpac CI 4</h1>

<p align="center">
<img src="https://webpac.lib.itb.ac.id/assets/images/webpac%202.png">
</p>

## Dokumentasi
  Progress Upgrade webpac [cek](https://github.com/dikper-dev/Dokumentasi-Upgrade-Webpac-CI-4).

## Project setup

```
lihat .env
```

Rubah Tabel staff_privileges dan ci_sessions

```
ALTER TABLE `staff_privileges` CHANGE `fileAccessed` `fileAccessed` TEXT CHARACTER SET latin1 COLLATE latin1_swedish_ci NULL DEFAULT NULL;
INSERT INTO `staff_privileges` (`privilegesID`, `staffGroupID`, `fileAccessed`) VALUES
(NULL, '41', 'all'),
(NULL, '1', 'dashboard;reloan;sirkulasi;bibliografi;ill;matakuliah;usulan;procurement;pengguna;other;');
```

```
DROP TABLE IF EXISTS `ci_sessions`;
CREATE TABLE `ci_sessions` (
`id` varchar(128) NOT NULL,
`ip_address` varchar(45) NOT NULL,
`timestamp` timestamp NOT NULL DEFAULT current_timestamp(),
`data` blob NOT NULL,
PRIMARY KEY (`id`,`ip_address`),
UNIQUE KEY `id_UNIQUE` (`id`),
KEY `ci_sessions_timestamp` (`timestamp`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

### Template Layout

Template baru untuk dashboard admin Metronomic
Dokumntasi [here](https://preview.keenthemes.com/html/metronic/docs/index).

- **Templating view** 
  ```html
  <!doctype html>
  <html>
  <head>
      <title><?= $this->renderSection('page_title', true) ?></title>
  </head>
  <body>
      <?= $this->extend('index') ?>

      <?= $this->section('content') ?>
        <h1>Hello World!</h1>
        <?= $this->include('sidebar') ?>
      <?= $this->endSection() ?>
  </body>
  </html>
  ```

- **Active View** 
  - view_cell, Dokumentasi [bisi lupa](https://codeigniter4.github.io/userguide/outgoing/view_cells.html).
  
  ```php
  //controller
  $dataDetail = ['activeView' => 'anggota/widgets/history_loan', 'data' => $sendData];

  //view
  if (!empty($dataDetail)) {
     echo view_cell('WidgetsCell::tampil', $dataDetail);
  }
  ```

- **Snippet Theme and function** 
  - Helpers `[theme_helper.php]` `[union_helper.php]`

- **Pagination** 
  ```php
  //model
  function loanHistory($userAccount, $page, $perPage)
  {
      $startFrom = ($page - 1) * $perPage;
      $sql = "SELECT SQL_CALC_FOUND_ROWS MAX(library.libraryTitle) as libraryTitle ---
            LIMIT " . $startFrom . ", " . $perPage . "";
      $query = $this->db->query($sql);
      $totalRowsQuery = $this->db->query("SELECT FOUND_ROWS() as totalRows");
      $totalRows = $totalRowsQuery->getRow()->totalRows;

      return [
          'results' => $query,
          'totalRows' => $totalRows
      ];
  }

  //controller
  $page = (int) ($this->request->getGet('page') ?? 1);
  $resloanhostory = $this->membermodel->loanHistory($userAccount, $page, $this->perPage);
  $pager_links = $this->pager->makeLinks($page, $this->perPage, $resloanhostory['totalRows'], 'temp_pager');
  $sendData = ['dataDetail' => $resloanhostory['results'], 'perPage' => $this->perPage, 'total' => $resloanhostory['totalRows'], 'pager_links' => $pager_links];
  $dataDetail = ['activeView' => 'anggota/widgets/history_loan', 'data' => $sendData];

  //view
  $headTable = ['No' => 'min-w-10px', 'Judul' => 'min-w-300px', 'Pinjam' => 'min-w-100px', 'Harus Kembali' => 'min-w-100px', 'Kembali' => 'min-w-100px', 'Perpustakaan' => 'min-w-100px'];
  $page = isset($_GET['page']) ? $_GET['page'] : 1;
  $iHis = 1 + ($perPage * ($page - 1));
  echo beginTableHead($headTable, 'HistoryLoanMember');
  foreach ($dataDetail->getResultArray() as $rHL) :
    $iHis++
    ---
  endforeach;
  echo endTableHead();
  <div class="d-flex flex-stack flex-wrap pt-3">
        <div class="fs-7 fw-bold text-gray-600">Showing <?= 1 + ($perPage * ($page - 1)) . ' to ' . $iHis - 1 . ' of ' . $total ?> entries</div>
        <?= $pager_links ?>
  </div>
  ```

### Path Config

- **SSO ITB** 
  - Config `[CasCon.php]`
  - Libraries `phpCAS-1.3.7` `[Cas.php]`

- **Email Config** 
  - Config `[Email.php]`
    
- **Autentifikasi Login dan Modul Akses** 
  - Config `[Filters.php]`
  - Filters `[AuthGuard.php]` `[RedirectIfAuthGuard.php]`

- **All Config Structure**
  - Config `[Union.php]`
