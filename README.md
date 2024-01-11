<h1 align="center">Ngeureuyeuh Webpac CI 4</h1>

<p align="center">
<img src="https://webpac.lib.itb.ac.id/assets/images/webpac%202.png">
</p>

## Dokumentasi
  Selalu cek [Mind Map](https://github.com/dikper-dev/webpac/blob/main/document/Mind-Map.md)
  ```
  hijau biru :: done
  kuning ada :: catatan
  merah on :: progress
  ```

  Progress Upgrade webpac [cek](https://github.com/dikper-dev/Dokumentasi-Upgrade-Webpac-CI-4).

## Ingat
  - Naikin performa website

    :: Kalo perlu gunakan caching atau session

  - Tuning Query

    :: Setiap nayamain model sama webpac yang lama, cek dulu yang sekiranya berat bikin model baru. gpp banyak model yang penting select sesuai kebutuhan

    :: Penamaan model sesuaikan dengan job atau aksinya biar ketauan relasinya

## Project setup

```
lihat .env
jika diperlukan untuk mysql terbaru 
https://stackoverflow.com/questions/23921117/disable-only-full-group-by
format 0000-00-00 00:00:00 error
https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_no_zero_date
FIX variable mysql mode STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
//NO_ZERO_IN_DATE,NO_ZERO_DATE,
```

Rubah Tabel staff_privileges dan ci_sessions

```mysql
ALTER TABLE `staff_privileges` CHANGE `fileAccessed` `fileAccessed` TEXT CHARACTER SET latin1 COLLATE latin1_swedish_ci NULL DEFAULT NULL;
INSERT INTO `staff_privileges` (`privilegesID`, `staffGroupID`, `fileAccessed`) VALUES
(NULL, '41', 'all'),
(NULL, '1', 'dashboard;reloan;sirkulasi;bibliografi;ill;matakuliah;usulan;procurement;pengguna;other;');
```

```mysql
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

Tambahan field table library sesuai RDA by Bu Esha

```mysql
ALTER TABLE `library` ADD `libraryProduction` VARCHAR(255) CHARACTER SET latin1 COLLATE latin1_swedish_ci NULL DEFAULT NULL AFTER `libraryYearPublished`, ADD `libraryDistribution` VARCHAR(255) CHARACTER SET latin1 COLLATE latin1_swedish_ci NULL DEFAULT NULL AFTER `libraryProduction`, ADD `libraryCreation` VARCHAR(255) CHARACTER SET latin1 COLLATE latin1_swedish_ci NULL DEFAULT NULL AFTER `libraryDistribution`;
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
  //or
  function daftarLib($cari, $ownerID, $order, $page, $perPage)
  {
      $startFrom = ($page - 1) * $perPage;
      $matchWhere = 'MATCH(libraryTitle,librarySubTitle) AGAINST("' . $cari . '" IN NATURAL LANGUAGE MODE) ';
      $matchSelect = $matchWhere . ' AS scores';

      if ($cari != '') {
          $sql = 'SELECT libraryID, libraryCallNumber,libraryISBN, libraryAuthor, libraryAdditionalAuthor, libraryTitle, librarySubTitle, libraryPublisher, libraryYearPublished, libraryImagePath, libraryImageName, libraryImageLink, ownerID, ' . $matchSelect;
      } else {
          $sql = 'SELECT libraryID, libraryCallNumber,libraryISBN, libraryAuthor, libraryAdditionalAuthor, libraryTitle, librarySubTitle, libraryPublisher, libraryYearPublished, libraryImagePath, libraryImageName, libraryImageLink, ownerID ';
      }

      $sql .= ' FROM library ';
      $sql .= ($cari != '') ?
          ' WHERE ' . $matchWhere . ' AND ownerID = ' . $ownerID :
          ' WHERE ownerID = ' . $ownerID;
      $sql .= ($order != '') ? ' ORDER BY ' . $order : ' ORDER BY libraryID DESC';
      $sql .= ' LIMIT ' . $startFrom . ", " . $perPage;
      $q = $this->db->query($sql);

      $totalRows = $this->db->table('library')
          ->select('COUNT(*) as totalRows')
          ->where(($cari != '') ? $matchWhere . ' AND ownerID = ' . $ownerID : 'ownerID = ' . $ownerID)
          ->get()
          ->getRow()->totalRows;

      return [
          'results' => $q,
          'totalRows' => $totalRows
      ];
  }
  //OR
  function weedingList($ownerID, $page, $perPage)
  {
      if ($page != '' && $perPage != '') {
          $startFrom = ($page - 1) * $perPage;
      }
      $sql = 'SELECT weeding.weedingDate, weeding.weedingDesc, library.libraryTitle, library_dataunit.libraryMainNumber, staff.staffName, library.libraryID  
          FROM weeding
          JOIN library_dataunit ON library_dataunit.libraryMainNumber = weeding.libraryMainNumber
          LEFT JOIN library ON library.libraryID = library_dataunit.libraryID
          JOIN staff ON staff.staffID = weeding.weedingStaffID ';
      $sql .= ' WHERE weeding.weedingStaffID > 0 AND weeding.weedingStaffDelete = 0 AND library_dataunit.ownerID = ' . $ownerID . ' AND weeding.ownerID = ' . $ownerID;
      $sql .= ' ORDER BY weedingDate DESC ';
      if ($page != '' && $perPage != '') {
          $sql .= 'LIMIT ' . $startFrom . ", " . $perPage;
      }
      $q = $this->db->query($sql);
      return ['results' => $q];
  }

  //controller
  $page = (int) ($this->request->getGet('page') ?? 1);
  $resloanhostory = $this->membermodel->loanHistory($userAccount, $page, $this->perPage);
  $pager_links = $this->pager->makeLinks($page, $this->perPage, $resloanhostory['totalRows'], 'temp_pager');
  $sendData = ['dataDetail' => $resloanhostory['results'], 'perPage' => $this->perPage, 'total' => $resloanhostory['totalRows'], 'pager_links' => $pager_links];
  $dataDetail = ['activeView' => 'anggota/widgets/history_loan', 'data' => $sendData];

  //ATAU
  $page = (int) ($this->request->getGet('page') ?? 1);
  $res_w = $this->wmodel->weedingList($this->session->ownerID, $page, $this->perPage);
  $rows = $this->getCachedRowsWeeding();
  $pager_links = $this->pager->makeLinks($page, $this->perPage, $rows, 'temp_pager');
  // :: biar ga diload terus
  private function getCachedRowsWeeding() //done
  {
      $sessionKey = 'cachedRowsWeeding';
      if ($this->session->has($sessionKey)) {
            return $this->session->get($sessionKey);
      } else {
            $whereRows = [
                'ownerID' => $this->session->ownerID,
                'weedingStaffID >' => 0,
                'weedingStaffDelete =' => 0
            ];
            $rows = $this->allModel->getTable('weeding', $whereRows)->getNumRows();
            $this->session->set($sessionKey, $rows);
            $this->session->markAsTempdata($sessionKey, 300);
            return $rows;
      }
  }
  // :: jika ada proses baru
  $this->session->remove('cachedRowsWeeding');

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
  - Libraries `[SendMail.php]`
  ```php
  // controller
  function loanMail($userAccount)
  {
      $print_user = $this->loanmodel->myLoan($this->session->ownerID, $userAccount);
      $message = '
      <table class="table gs-7 gy-7 gx-7">
          <thead>
              <tr class="fw-bold fs-6 text-gray-800 border-bottom border-gray-200">
                  <th>No</th>
                  <th>Barcode</th>
                  <th>Buku</th>
                  <th>Tanggal Pinjam</th>
                  <th>Harus Kembali</th>
              </tr>
          </thead>';
      $i = 1;
      foreach ($print_user->getResult() as $r) {
          $message .= '
                  <tr>
                      <td>' . $i++ . '</td>
                      <td>' . $r->libraryMainNumber . '</td>
                      <td>' . $r->libraryTitle . '</td>			
                      <td>' . date_to_tanggal($r->loanDate) . '</td>	
                      <td>' . date_to_tanggal($r->returnDate) . '</td>	
                  </tr>
              ';
      }
      '</table>';
      // echo $message; //check data
      $sendMail = new SendMail();
      $pesan = $sendMail->send($userAccount, 'BUKTIPINJAM', 'Bukti Peminjaman Buku', $message);
      return redirect()->back()->with('pesan', $pesan);
  }
  ```
    
- **Autentifikasi Login dan Modul Akses** 
  - Config `[Filters.php]`
  - Filters `[AuthGuard.php]` `[RedirectIfAuthGuard.php]`

- **All Config Structure**
  - Config `[Union.php]`

**CATATAN saja**

*Belajar Tambahin SEO*
```
$decodedInput = urldecode((string) $cleanInput);
$q = url_title($decodedInput, '-', true);
stackoverflow
https://stackoverflow.com/questions/35241656/how-to-decode-url-title-function-in-codeigniter
:: segment/id/url_title{slug}

Amazone
search: Papoea's waren mijn makkers
https://www.amazon.com/Papoeas-Waren-Mijn-Makkers-Exploratietocht/dp/B00EDS687Y
:: url_title{slug_title}/detail_page/id
or
https://www.amazon.com/-/es/Eric-Lundquist/dp/B00EDS687Y
:: {localization}/url_title{slug_author}/detail_page/id
```

```
apakah view cell itu 1 control 1 fungsi ?
<?= view_cell('WidgetsCell::tampil', $dataDetail) ?>
:: Langsung call aja view utama nya, tinggal activeView sma data nya. contoh nya di bibliografi

//perbedaan <?= dan <?php
<?= base_url() ?> => reset path url
<?php base_url() ?> => url active path

redirect()->back()->with() harus proses aksi tanpa view
```
```phph
//APA INI ?????
UPDATE `ci_sessions` SET `timestamp` = now(), `data` = '__ci_last_regenerate|i:1700467826;_ci_previous_url|s:75:\"http://localhost:8080/pengguna/anggota/search?q=ariftrikanda&detail=on-loan\";userID|s:3:\"201\";itbAccount|s:19:\"dikdikper@itb.ac.id\";userLogin|s:19:\"dikdikper@itb.ac.id\";userPassword|s:9:\"dikdikper\";userName|s:14:\"Dikdik Permana\";userPhoto|s:40:\"./assets/files/photo_staff/dikdikper.jpg\";userEmail|s:19:\"dikdikper@itb.ac.id\";userITBNumber|s:8:\"10061992\";logged_in|b:1;loginversi|s:3:\"cas\";ownerID|s:1:\"1\";ownerAllowNIM|s:1:\"*\";ownerInitial|s:5:\"PUSAT\";ownerActiveFine|s:1:\"1\";ownerHoliday|s:1:\"6\";userModulAccess|s:3:\"all\";userGroupID|s:2:\"41\";ipPrinter|s:12:\"167.205.4.22\";orientationPolicy|s:1:\"1\";dateLastLogin|s:19:\"2023-11-20 01:13:00\";year|s:4:\"2023\";groupName|s:9:\"superuser\";prodyID|s:4:\"7807\";multiAgent|s:1:\"1\";kodeKampus|s:0:\"\";campusCode|s:2:\"JN\";statAktif|s:0:\"\";noRegistration|s:0:\"\";uaID|s:6:\"126502\";member|s:8:\"10061992\";memberType|s:6:\"Tendik\";disabledInput|s:0:\"\";memberStatus|s:4:\"NULL\";memberSiskeu|s:4:\"NULL\";statusPinjam|s:0:\"\";prodyCode|s:0:\"\";prody|s:0:\"\";data_user|O:8:\"stdClass\":42:{s:6:\"userID\";s:6:\"126502\";s:12:\"departmentID\";N;s:8:\"userType\";s:1:\"3\";s:11:\"userAccount\";s:8:\"10061992\";s:10:\"itbAccount\";s:19:\"dikdikper@itb.ac.id\";s:8:\"isActive\";s:1:\"1\";s:10:\"isApproved\";s:1:\"0\";s:8:\"userName\";s:14:\"Dikdik Permana\";s:11:\"userAddress\";s:7:\"Bandung\";s:9:\"userPhone\";s:12:\"082126360696\";s:9:\"userEmail\";s:19:\"dikdikper@itb.ac.id\";s:12:\"userEmailITB\";s:22:\"mynamedikdik@gmail.com\";s:10:\"userExpire\";s:10:\"0000-00-00\";s:13:\"userImageName\";s:0:\"\";s:13:\"userImageType\";s:0:\"\";s:13:\"userImageSize\";s:0:\"\";s:13:\"staffInsertID\";s:3:\"201\";s:15:\"staffInsertDate\";s:19:\"2021-06-28 09:32:41\";s:11:\"staffEditID\";N;s:13:\"staffEditDate\";s:19:\"0000-00-00 00:00:00\";s:6:\"rootID\";s:9:\"100619921\";s:17:\"userOrientationed\";s:1:\"1\";s:19:\"userOrientationDate\";s:19:\"0000-00-00 00:00:00\";s:21:\"orientationDeleteDate\";s:19:\"0000-00-00 00:00:00\";s:19:\"staffOrientationAdm\";s:1:\"0\";s:23:\"staffOrientationTeacher\";s:1:\"0\";s:24:\"userOrientationedSuccess\";s:1:\"0\";s:20:\"userOrientationScore\";s:1:\"0\";s:15:\"userHasFreeLoan\";s:1:\"0\";s:16:\"userFreeLoanDate\";N;s:19:\"userFreeLoanStaffID\";s:1:\"0\";s:17:\"userHasGraduation\";s:1:\"0\";s:21:\"userHasGraduationDate\";N;s:24:\"userHasGraduationStaffID\";s:3:\"201\";s:23:\"userRegisterationNumber\";s:1:\"-\";s:13:\"officeAccount\";s:26:\"dikdikper@office.itb.ac.id\";s:7:\"userINA\";s:9:\"dikdikper\";s:9:\"prodiCode\";s:1:\"-\";s:10:\"campusCode\";s:2:\"JN\";s:13:\"six_stataktif\";s:1:\"-\";s:14:\"six_tgl_wisuda\";N;s:13:\"six_tgl_lulus\";N;}onLoanTPB|a:1:{i:0;O:8:\"stdClass\":50:{s:11:\"libraryDate\";s:19:\"2014-12-15 10:34:59\";s:12:\"staffAccount\";s:3:\"153\";s:9:\"libraryID\";s:6:\"111781\";s:19:\"libraryMaterialType\";s:2:\"19\";s:17:\"libraryCollection\";s:2:\"17\";s:17:\"libraryCallNumber\";s:11:\"530.078 KIR\";s:11:\"libraryISBN\";s:68:\"ISBN-10: 0471335797 [paperback] ; ISBN-13: 9780471335795 [paperback]\";s:13:\"libraryAuthor\";s:11:\"KIRKUP, Les\";s:23:\"libraryAdditionalAuthor\";s:0:\"\";s:16:\"libraryCooperate\";s:0:\"\";s:12:\"libraryTitle\";s:79:\"Experimental methods : an introduction to the analysis and presentation of data\";s:15:\"librarySubTitle\";s:16:\"/ by Les Kirkup.\";s:14:\"libraryEdition\";s:0:\"\";s:20:\"libraryCityPublished\";s:11:\"Milton, Qld\";s:16:\"libraryPublisher\";s:5:\"Wiley\";s:20:\"libraryYearPublished\";s:4:\"1994\";s:13:\"libraryVolume\";s:0:\"\";s:11:\"libraryDesc\";s:0:\"\";s:10:\"libraryUrl\";s:0:\"\";s:13:\"librarySeries\";s:0:\"\";s:14:\"librarySubject\";s:16:\"Physics ; Fisika\";s:14:\"libraryKeyword\";s:161:\"Physics -- laboratory manuals, physics -- experiments, physics -- data processing ; Fisika -- manual laboratorium, fisika -- percobaan, fisika -- pengolahan data\";s:18:\"libraryIsReference\";s:1:\"0\";s:11:\"libraryGain\";s:35:\"Beli (DIPA 2014 usulan FMIPA-FI-17)\";s:12:\"libraryNotes\";s:48:\"xv, 216 halaman : gambar, tabel ; 23 centimeter.\";s:16:\"libraryImagePath\";s:33:\"./assets/files/cover/old/2014/12/\";s:16:\"libraryImageName\";s:10:\"111781.jpg\";s:16:\"libraryImageType\";s:10:\"image/jpeg\";s:16:\"libraryImageSize\";s:5:\"31135\";s:16:\"libraryImageLink\";N;s:15:\"libraryEditDate\";s:19:\"2015-06-01 10:29:50\";s:16:\"staffaccountEdit\";s:3:\"153\";s:15:\"libraryLanguage\";s:1:\"2\";s:16:\"libraryID_Source\";N;s:7:\"ownerID\";s:1:\"1\";s:6:\"loanID\";s:6:\"774039\";s:17:\"libraryMainNumber\";s:9:\"201408104\";s:10:\"returnDate\";s:10:\"2023-06-21\";s:8:\"loanDate\";s:10:\"2023-06-07\";s:4:\"hari\";s:3:\"152\";s:2:\"id\";s:2:\"17\";s:14:\"collectionName\";s:16:\"Koleksi Mingguan\";s:15:\"collectionLabel\";s:0:\"\";s:7:\"loanDay\";s:2:\"14\";s:14:\"collectionFine\";s:4:\"1000\";s:6:\"active\";s:1:\"1\";s:13:\"staffInsertID\";N;s:10:\"insertDate\";N;s:11:\"staffEditID\";s:2:\"78\";s:8:\"editDate\";s:19:\"2020-06-24 14:39:38\";}}onCaseTPB|a:1:{i:0;O:8:\"stdClass\":17:{s:10:\"userCaseID\";s:4:\"1559\";s:11:\"userAccount\";s:9:\"100619921\";s:12:\"userCaseDesc\";s:32:\"Test Kasus untuk mobile aplikasi\";s:12:\"userCaseDate\";s:10:\"2022-03-04\";s:10:\"caseTypeID\";N;s:10:\"caseDetail\";s:0:\"\";s:14:\"userCaseAmount\";s:1:\"0\";s:13:\"staffInsertID\";s:3:\"201\";s:16:\"userCaseDateEdit\";s:10:\"0000-00-00\";s:11:\"staffEditID\";s:1:\"0\";s:10:\"caseClosed\";s:1:\"0\";s:15:\"staffCaseClosed\";s:3:\"201\";s:14:\"dateCaseClosed\";s:10:\"2022-06-23\";s:7:\"ownerID\";s:1:\"1\";s:7:\"loanIDS\";N;s:8:\"userName\";s:14:\"Dikdik Permana\";s:9:\"staffName\";s:14:\"Dikdik Permana\";}}' WHERE `id` = 'ci_session:upvke61l1995oos3gv5u2ke3lddi557c'

0.25 ms	SELECT RELEASE_LOCK('3ad93d6308ed9b2f39045194876e9a52') AS ci_session_lock	[internal function]
0.26 ms	SELECT GET_LOCK('fb502c3b23bb21ed25c60b008ed2f48a', 300) AS ci_session_lock

21.41 ms	SELECT `u`.`userAccount`, `u`.`itbAccount`, `u`.`userName`, `u`.`userEmail`, `u`.`userEmailITB`, `u`.`userAddress`, `u`.`userINA`, `u`.`userPhone`, `u`.`campusCode`, `u`.`userType`, `u`.`rootID`, `u`.`officeAccount`, `u`.`userOrientationed`, `t`.`typeName`, `t`.`typeCss` FROM `user_account` `u` LEFT JOIN `user_type` `t` ON `t`.`id` = `u`.`userType` WHERE `userAccount` = 'ariftrikanda' GROUP BY `u`.`userAccount`	APPPATH\Models\Pengguna\MemberModel.php:34
917.47 ms	SELECT `u`.`userAccount`, `u`.`itbAccount`, `u`.`userName`, `u`.`userEmail`, `u`.`userEmailITB`, `u`.`userAddress`, `u`.`userINA`, `u`.`userPhone`, `u`.`campusCode`, `u`.`userType`, `u`.`rootID`, `u`.`officeAccount`, `u`.`userOrientationed`, `t`.`typeName`, `t`.`typeCss` FROM `user_account` `u` LEFT JOIN `user_type` `t` ON `t`.`id` = `u`.`userType` WHERE `u`.`userAccount` LIKE '%ariftrikanda%' ESCAPE '!' OR `u`.`userINA` LIKE '%ariftrikanda%' ESCAPE '!' OR `u`.`userName` LIKE '%ariftrikanda%' ESCAPE '!' GROUP BY `u`.`userAccount`

43.04 ms	SELECT `u`.`userAccount`, `u`.`itbAccount`, `u`.`userName`, `u`.`userEmail`, `u`.`userEmailITB`, `u`.`userAddress`, `u`.`userINA`, `u`.`userPhone`, `u`.`campusCode`, `u`.`userType`, `u`.`rootID`, `u`.`officeAccount`, `u`.`userOrientationed`, `t`.`typeName`, `t`.`typeCss` FROM `user_account` `u` LEFT JOIN `user_type` `t` ON `t`.`id` = `u`.`userType` WHERE `userAccount` = '10023116' GROUP BY `u`.`userAccount`
```
