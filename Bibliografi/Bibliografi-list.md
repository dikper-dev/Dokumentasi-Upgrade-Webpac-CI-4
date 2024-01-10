# Bibliografi List
## Modul

Tambahan field sesuai RDA by Bu Esha
```mysql
ALTER TABLE `library` ADD `libraryProduction` VARCHAR(255) CHARACTER SET latin1 COLLATE latin1_swedish_ci NULL DEFAULT NULL AFTER `libraryYearPublished`, ADD `libraryDistribution` VARCHAR(255) CHARACTER SET latin1 COLLATE latin1_swedish_ci NULL DEFAULT NULL AFTER `libraryProduction`, ADD `libraryCreation` VARCHAR(255) CHARACTER SET latin1 COLLATE latin1_swedish_ci NULL DEFAULT NULL AFTER `libraryDistribution`;
```

Takutnya error jaga2

```mysql
ALTER TABLE `library` CHANGE `libraryDate` `libraryDate` DATETIME DEFAULT NULL;
ALTER TABLE `library` CHANGE `libraryEditDate` `libraryEditDate` DATETIME DEFAULT NULL;
```

NOTE :: Banyak duplicate entry jika input buku

- **Search** [&#10003;]

*ADA BUGS*
```php
//jika diruning di server jalan, di versi mysql 8.2 ga jalan
SELECT `libraryID`, `libraryCallNumber`, `libraryISBN`, `libraryAuthor`, `libraryAdditionalAuthor`, `libraryTitle`, `librarySubTitle`, `libraryPublisher`, `libraryYearPublished`, `libraryImagePath`, `libraryImageName`, `libraryImageLink`, `ownerID`, MATCH(libraryTitle, librarySubTitle) AGAINST("itb" IN NATURAL LANGUAGE MODE) AS scores FROM `library` WHERE MATCH(libraryTitle,librarySubTitle) AGAINST("itb" IN NATURAL LANGUAGE MODE) AND `ownerID` = '1' ORDER BY `scores` DESC
```
more improvement di pencarian url
```js
<script language="JavaScript" type="text/javascript">
    document.getElementById('fSearchBiblio').addEventListener('submit', function(e) {
        e.preventDefault();
        let formAction = this.getAttribute('action');
        formAction = formAction.replace(/&?bCari=bCari/, '');
        this.setAttribute('action', formAction);
        let inputValue = document.getElementById('q').value.trim();
        let cleanedString = inputValue.replace(/[^\w\säöü-]/gi, '').replace(/\s+/g, '-');
        document.getElementById('q').value = cleanedString;
        this.submit();
    });
</script>
```
- **Daftar Bibliografi** [&#10003;]

from
```php
//controller
public function index($pesan = '', $page = '')
{
     $data['login'] = $this->auth->cekLOGIN($this->modul);

     if ($data['login']) {
          $this->load->library('pagination');
          $data['BASE_CTRL'] = $this->base_ctrl;
          $data['title'] = $this->lang->line('bibliografi');
          $data['activeView'] = $this->base_view . '/v_bibliografi_list';
          if ($pesan != '') {
               $data['pesan'] = $pesan;
          } else {
               $data['pesan'] = '';
          }

          //paging
          isset($_GET['per_page']) ? $page = $_GET['per_page'] : $page = 0;
          isset($_GET['rows']) ? $rows = $_GET['rows'] : $rows = null;

          $where = '';
          $cari = $this->input->get('tCari', true);

          //!Dikdik Testing
          $cari = str_replace("'", "", $cari);
          $cari = str_replace('"', '', $cari);

          // var_dump($this->escape_str($cari, TRUE));die;

          if (trim($cari) != '') {
               // $q = $this->escape_str($cari, TRUE); //! ESCAPE "
               $q = $cari;
               $order = 'scores DESC';
               $paramGet = $this->base_ctrl . '/index?tCari=' . $q . '&bCari=' . $_GET['bCari'] . '&';
               if ($rows == null) {
               $limit = '';
               } else {
               $limit = $this->perPage . ',' . $page;
               }
          } else {
               $paramGet = $this->base_ctrl . '/index?';
               $q = '';
               $order = '';
               if ($rows == null) {
               $this->db->where('ownerID', $this->ownerID);
               $rows = $this->db->count_all_results('library');
               }
               // echo $this->db->last_query().'<hr>';
               $limit = $this->perPage . ',' . $page;
          }

          // echo 'limit='.$limit;
          $daftarLib = $this->bibmodel->daftarLib($q, $this->ownerID, $order, $limit);
          if ($rows == null) {
               $rows = $daftarLib->num_rows();
          }
          // echo $this->db->last_query().'<hr>';

          $paramGet .= 'rows=' . $rows;
          $configPage = $this->atfungsi->configPage2($paramGet, $rows, $this->perPage);
          $data['bibliografi'] = $daftarLib;

          $this->pagination->initialize($configPage);
          $data['page_link'] = $this->pagination->create_links();
          $data['page'] = $page;
          $data['perPage'] = $this->perPage;
          $data['rows'] = $rows;
          $data['pesan'] = array('', '');
          $data['cari'] = $q;

          $this->load->view('v_home', $data);
     } else {
          redirect(base_url());
     }
}
//model
function daftarLib($cari, $ownerID, $order = '', $limit = '')
{
     $matchWhere = 'MATCH(libraryTitle,librarySubTitle) AGAINST("'.$cari.'" IN NATURAL LANGUAGE MODE)';
     $matchSelect = $matchWhere.' AS scores';
     
     if($cari!=''){
          $this->db->select($this->SELECT_FIELDS.','.$matchSelect);
          $this->db->where($matchWhere);
     }
     else{
          $this->db->select($this->SELECT_FIELDS);
     }

     $this->db->where('ownerID', $ownerID);
     
     if($order!=''){
          $this->db->order_by($order);
     }
     else{
          $this->db->order_by('libraryID DESC');
     }

     if($limit!=''){
          // $this->db->limit($this->perPage, $offset);
          $arrLimit = explode(',', $limit);
          $this->db->limit($arrLimit[0], $arrLimit[1]);
     }

     $res = $this->db->get('library');
     return $res;
}
```
to
```php
//controller
public function index()
{
     $cari = trim((string) $this->request->getGet('q'));
     if ($cari != '') {
          $order = 'scores DESC';
          $q = str_replace(["'", '"'], '', $cari);
     } else {
          $order = $q = '';
     }

     $page = (int) ($this->request->getGet('page') ?? 1);
     $daftarLib = $this->bibmodel->daftarLib($q, $this->session->ownerID, $order, $page, $this->perPage);
     $pager_links = $this->pager->makeLinks($page, $this->perPage, $daftarLib['totalRows'], 'temp_pager');
     $sendData = [
          'headTable' => ['No' => 'min-w-10px', 'Bibilografi' => 'min-w-400px', 'Penulis' => 'min-w-300px', 'Aksi' => 'min-w-50px'],
          'bibliografi' => $daftarLib['results'],
          'perPage' => $this->perPage,
          'total' => $daftarLib['totalRows'],
          'pager_links' => $pager_links
     ];
     $data['dataDetail'] = ['activeView' => 'bibliografi/partials/biblio_list', 'data' => $sendData];
     $data['cari'] = str_replace('-', ' ', $cari);
     $data['BASE_CTRL'] = $this->base_ctrl;
     $data['title'] = 'Daftar Bibliografi';
     return view('bibliografi/biblio', $data);
}
//model
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

     //NOTE 
     // kalo pake SELECT SQL_CALC_FOUND_ROWS
     // $totalRowsQuery = $this->db->query("SELECT FOUND_ROWS() as totalRows");
     // $totalRows = $totalRowsQuery->getRow()->totalRows;
     // tapi lebih lambat beberapa ms kalo pake ini
     // di model staf aman sih, cmn datanya lebih sedikit
     // ini lebih lambat kalo pake selectCount ci4
     // $totalRows = $this->db->table('library')
     //     ->selectCount('libraryID')
     //     ->where(($cari != '') ? $matchWhere . ' AND ownerID = ' . $ownerID : 'ownerID = ' . $ownerID)
     //     ->get()
     //     ->getRow()->libraryID;
}
```
- **Delete Bibliografi** [&#10003;]

*Proses Hapus* [&#10003;]

from
```php
function deletelib()
{
     $data['login'] = $this->auth->cekLOGIN($this->modul);
     if ($data['login']) {
          $ownerID = $this->session->userdata('ownerID');
          $libraryID = $this->input->post('liby');
          $libTitle = $this->input->post('libx');

          //get library
          $this->db->select('library.*');
          $this->db->where('library.libraryID', $libraryID);
          $yx = $this->db->get('library')->row();
          $data = array(
               'libraryDate' => $yx->libraryDate,
               'staffAccount' => $yx->staffAccount,
               'libraryID' => $yx->libraryID,
               'libraryMaterialType' => $yx->libraryMaterialType,
               'libraryCollection' => $yx->libraryCollection,
               'libraryCallNumber' => $yx->libraryCallNumber,
               'libraryISBN' => $yx->libraryISBN,
               'libraryAuthor' => $yx->libraryAuthor,
               'libraryAdditionalAuthor' => $yx->libraryAdditionalAuthor,
               'libraryCooperate' => $yx->libraryCooperate,
               'libraryTitle' => $yx->libraryTitle,
               'librarySubTitle' => $yx->librarySubTitle,
               'libraryEdition' => $yx->libraryEdition,
               'libraryCityPublished' => $yx->libraryCityPublished,
               'libraryPublisher' => $yx->libraryPublisher,
               'libraryYearPublished' => $yx->libraryYearPublished,
               'libraryVolume' => $yx->libraryVolume,
               'libraryDesc' => $yx->libraryDesc,
               'libraryUrl' => $yx->libraryUrl,
               'librarySeries' => $yx->librarySeries,
               'librarySubject' => $yx->librarySubject,
               'libraryKeyword' => $yx->libraryKeyword,
               'libraryIsReference' => $yx->libraryIsReference,
               'libraryGain' => $yx->libraryGain,
               'libraryNotes' => $yx->libraryNotes,
               'libraryImageName' => $yx->libraryImageName,
               'libraryImageLink' => $yx->libraryImageLink,
               'libraryEditDate' => $yx->libraryEditDate,
               'staffaccountEdit' => $yx->staffaccountEdit,
               'libraryLanguage' => $yx->libraryLanguage
          );
          $res1 = json_encode($data);

          //get datun
          $this->db->select('library_dataunit.*');
          $this->db->where('library_dataunit.libraryID', $libraryID);
          $this->db->where('library_dataunit.ownerID', $ownerID);
          $xy = $this->db->get('library_dataunit')->result();
          $data = array();
          foreach ($xy as $key) {
               $row = array();
               $row[] = $key;
               $data[] = $row;
          }
          $res2 = json_encode($data);

          //get lib owner
          $this->db->select('library_owner.*');
          $this->db->where('library_owner.libraryID', $libraryID);
          $wy = $this->db->get('library_owner')->result();
          $datao = array();
          foreach ($wy as $key) {
               $row = array();
               $row[] = $key;
               $datao[] = $row;
          }
          $res3 = json_encode($datao);

          //proses delete library
          $this->db->where('libraryID', $libraryID);
          $this->db->delete('library');

          //library_owner delete 
          $this->db->where('libraryID', $libraryID);
          $this->db->delete('library_owner');

          //proses delete datun
          $this->db->where('library_dataunit.libraryID', $libraryID);
          $this->db->where('library_dataunit.ownerID', $ownerID);
          $this->db->delete('library_dataunit');

          //proses insert log
          $datasave['logLibrary'] = $res1;
          $datasave['logDatun'] = $res2;
          $datasave['logLibraryOwner'] = $res3;
          $this->db->insert('library_delete_log', $datasave);

          $pesan = array('Berhasil hapus library ' . $libTitle, 'warning');
          $this->index($pesan);
     } //end login
     else {
          redirect(base_url());
     }
}
```
to
```php
//ada tambahan field di libart_delete_log
public function delete()
{
     $ownerID = $this->session->ownerID;
     $libraryID = $this->request->getPost('liby');
     $libTitle = $this->request->getPost('libx');

     // Get all data by id and owner
     $libraryDetails = $this->allModel->getTable('library', ['libraryID' => $libraryID])->getRow();
     $libraryDataUnits = $this->allModel->getTable('library_dataunit', ['libraryID' => $libraryID, 'ownerID' => $ownerID])->getResult();
     $libraryOwnerDetails = $this->allModel->getTable('library_owner', ['libraryID' => $libraryID])->getResult();

     // Delete all data by id and owner
     $this->allModel->deleteTable('library', ['libraryID' => $libraryID]);
     $this->allModel->deleteTable('library_owner', ['libraryID' => $libraryID]);
     $this->allModel->deleteTable('library_dataunit', ['libraryID' => $libraryID, 'ownerID' => $ownerID]);

     // Save logs
     $logData = [
          'deleteStaff' => $this->session->userID,
          'deleteDate' => date('Y-m-d H:i:s'),
          'libraryTitle' => $libTitle, //Keperluan Jika Butuh Restore data setelah dihapus
          'logLibrary' => json_encode($libraryDetails),
          'logDatun' => json_encode($libraryDataUnits),
          'logLibraryOwner' => json_encode($libraryOwnerDetails),
     ];
     $this->allModel->add('library_delete_log', $logData);

     return redirect()->to($this->base_ctrl)->with('pesan', ['warning', 'Berhasil Hapus Data Bibliografi <b>' . $libTitle . '</b>']);
}
```

*Cek Data Jika Hapus Library* [&#10003;]

from
```php
function cekUnit()
{
     $ownerID = $this->session->userdata('ownerID');
     $libraryID = $this->input->post('libID');

     //select owner
     $perpus = $this->allmodel->getTable('owner', 'ownerID="' . $ownerID . '"')->row();
     $perpusnama = $perpus->ownerName;
     $msg = '';

     //cek datun where ownerID != session ownerID
     $cekDatunOnLoan = $this->settingmodel->table('library_dataunit', 'libraryID ="' . $libraryID . '" AND ownerID !="' . $ownerID . '" '); //!clear

     //cek onloan == loan ownerID
     $onLoan = $this->bibmodel->libraryOnLoan($libraryID, $ownerID); //!clear
     $jumlahOnLoan = $onLoan->num_rows();

     //cek unit
     $owners = $this->bibmodel->ownersBibliografi($libraryID, 'library_dataunit.ownerID ="' . $ownerID . '"')->row_array();
     $tot = $owners['eksemplar']; // - $jumlahOnLoan
     if ($tot > 0) {
          $mes = 'Buku ini memiliki <strong>' . $tot . ' Data Unit. Yakin Hapus ?</strong>';
     } else {
          $mes = 'Buku ini <strong> Tidak Memiliki Data Unit. Hapus ?</strong>';
     }

     //set kondisi
     if ($cekDatunOnLoan->num_rows() > 0) {
          $flag = '1';
          $msg .= 'Buku ini memiliki Data Unit selain di <strong>' . $perpusnama . '. Tidak Bisa Hapus Library! </strong>';
     } elseif ($jumlahOnLoan > 0) {
          $flag = '1';
          $msg .= ' Buku ini memiliki Data unit yang sedang dipinjam. <strong> Tidak Bisa Hapus Library! </strong> </br>';
     } else {
          $flag = '0';
          $msg = $mes; //!weeding perbaikan tampilkan disini
     }
     //output to json format
     $output = array(
          'flag' => $flag,
          'msg'  => $msg
     );
     echo json_encode($output);
}
```
to

contoh jika count dan tidak ada hasil tidak bisa langsung return 0, harus ->getNumRows() dlu tidak bisa langsung getRowArray() misalnya, kasus 2 ada di detail ketersediaan
```php
//controller
$owners = $this->bibmodel->delCheckOwnersBibliografi($libraryID, 'library_dataunit.ownerID ="' . $ownerID . '"')->getRowArray();
$tot = $owners['eksemplar']; //maka tidak bisa return 0
```

```php
//controller
function delete_check()
{
     $msg = '';
     $ownerID = $this->session->ownerID;
     $libraryID = $this->request->getPost('libID');

     $cekDatunOnLoan = $this->bibmodel->delCheckDatunOnLoan($libraryID, $ownerID); //cek datun where ownerID != session ownerID
     $onLoan = $this->bibmodel->delCheckLibraryOnLoan($libraryID, $ownerID); //cek onloan == loan ownerID

     if ($cekDatunOnLoan->getNumRows() > 0) {
          $flag = '1';
          $msg = 'Buku ini memiliki Data Unit selain di <strong>' . $cekDatunOnLoan->getRow()->ownerName . '. Tidak Bisa Hapus Library! </strong>';
     } elseif ($onLoan->getNumRows() > 0) {
          $flag = '1';
          $msg = ' Buku ini memiliki Data unit yang sedang dipinjam. <strong> Tidak Bisa Hapus Library! </strong> </br>';
     } else {
          $flag = '0';
          $res_owners = $this->bibmodel->delCheckOwnersBibliografi($libraryID, 'library_dataunit.ownerID ="' . $ownerID . '"'); //cek unit
          if ($res_owners->getNumRows() > 0) {
               $owners = $res_owners->getRowArray();
               $tot = $owners['eksemplar'];
               if ($tot > 0) {
                    $mes = 'Buku ini memiliki <strong>' . $tot . ' Data Unit. Yakin Hapus ?</strong>';
               }
          } else {
               $mes = 'Buku ini <strong> Tidak Memiliki Data Unit. Hapus ?</strong>';
          }
          $msg = $mes;
     }

     $output = array(
          'flag' => $flag,
          'msg'  => $msg
     );
     return $this->response->setJSON($output);
}
//model
function delCheckDatunOnLoan($libraryID, $ownerID)
{
     $b = $this->db->table('library_dataunit');
     $b->select('library_dataunit.libraryMainNumber, owner.ownerName');
     $b->join('owner', 'owner.ownerID = library_dataunit.ownerID');
     $b->where('library_dataunit.libraryID', $libraryID);
     $b->where('library_dataunit.ownerID !=', $ownerID);
     return $b->get();
}

function delCheckLibraryOnLoan($libraryID, $ownerID)
{
     $b = $this->db->table('loan');
     $b->select('MAX(loan.libraryMainNumber) as libraryMainNumber, MAX(loan.loanID) as loanID');
     $b->join('library_dataunit', 'library_dataunit.libraryMainNumber=loan.libraryMainNumber');
     $b->where('loan.returnedDate', '0000-00-00');
     $b->where('library_dataunit.libraryID', $libraryID);
     $b->where('library_dataunit.ownerID', $ownerID);
     $b->where('loan.ownerID', $ownerID);
     $b->groupBy('loan.libraryMainNumber');
     $b->orderBy('loan.libraryMainNumber', 'DESC');
     return $b->get();
}

function delCheckOwnersBibliografi($libraryID, $where = '', $owner = '')
{
     $b = $this->db->table('library');
     $b->select('count(library_dataunit.libraryMainNumber) as eksemplar');
     $b->join('library_dataunit', 'library_dataunit.libraryID=library.libraryID');
     $b->join('owner', 'owner.ownerID=library_dataunit.ownerID');
     $b->where('library.libraryID', $libraryID);
     if ($owner) {
          $b->where('ownerID', $owner);
     }
     if ($where != '') {
          $b->where($where);
     }
     $b->groupBy('library_dataunit.ownerID');
     return $b->get();
}
```
- **Input Bibliografi** [&#10003;]
```php
function add($pesan = '')
{
     $this->auth->cekLOGIN($this->modul);
     $data['pesan'] = $pesan;
     $data['login'] = $this->auth->cekLOGIN($this->modul);
     if ($data['login']) {
          $data['BASE_CTRL'] = $this->base_ctrl;
          $data['title'] = $this->lang->line('add') . ' ' . $this->lang->line('bibliografi');
          $data['viewBawah'] = '';

          $btnCek = $this->input->post('btnCek');
          if ($btnCek == 'Cek') {
               $isbn = trim($this->input->post('tISBN'));
               //cek di bibliografi sendiri
               $book = $this->allmodel->getTable('library', 'libraryISBN like "%' . $isbn . '%" AND ownerID="' . $this->session->userdata('ownerID') . '" LIMIT 5');
               if ($book->num_rows() > 0) { //buku sudah ada di webpac
               // echo 'buku sudah tersedia di webpac';
               $data['activeView'] = 'bibliografi/v_bibliografi_add_isbn';
               $data['viewBawah'] = 'bibliografi/v_bibliografi_add_isbn_list';
               $data['book'] = $book;
               } else { //buku belum ada di webpac

               // $data['viewBawah'] = '';
               $url = 'https://www.googleapis.com/books/v1/volumes?q=isbn:' . $isbn;

               /*
               // $ch = curl_init();
               // curl_setopt($ch, CURLOPT_HEADER, false);
               // curl_setopt($ch, CURLOPT_URL, $url);
               // curl_setopt($ch, CURLOPT_SSLVERSION, 1);
               // $result = curl_exec($ch);
               // curl_close($ch);
               // var_dump($result);die;
               // $contents = $this->getSSLPage($url);

               // $arrContextOptions=array(
               //     "ssl"=>array(
               //         "verify_peer"=>false,
               //         "verify_peer_name"=>false,
               //     ),
               // );  

               
               $contents = file_get_contents($url);
               // var_dump($contents);die;
               $arrContextOptions=array(
                    "http" => array(
                         "method" => "POST",
                         "header" => 
                              "Content-Type: application/xml; charset=utf-8;\r\n".
                              "Connection: close\r\n",
                         "ignore_errors" => true,
                         "timeout" => (float)30.0,
                         // "content" => $strRequestXML,
                    ),
                    "ssl"=>array(
                         "allow_self_signed"=>true,
                         "verify_peer"=>false,
                         "verify_peer_name"=>false,
                    ),
               );
               // $contents = file_get_contents($url, false, stream_context_create($arrContextOptions));
               $book = json_decode($contents);
               */
               echo $url;
               $book = $this->getBook($isbn);
               print_r($book);
               exit;

               $data['bookGoogle'] = $book;
               $data['googleURL'] = $url;
               $data['book'] = array();

               // echo $book->totalItems;
               if ($book->totalItems > 0) {
                    $subTitle = '';
                    $author2 = '';
                    $category = '';
                    $libISBN = '';
                    $countAuthor = count($book->items[0]->volumeInfo->authors);

                    if ($countAuthor > 1) {
                         $i = 1;
                         foreach ($book->items[0]->volumeInfo->authors as $rAuthor) {
                              $author2 .= $rAuthor . '; ';
                              $i++;
                         }
                    }

                    if (isset($book->items[0]->volumeInfo->subtitle)) {
                         $subTitle = $book->items[0]->volumeInfo->subtitle;
                    }

                    if (isset($book->items[0]->volumeInfo->categories[0])) {
                         $countCategory = count($book->items[0]->volumeInfo->categories);
                         if ($countCategory > 1) {
                              $i = 0;
                              foreach ($book->items[0]->volumeInfo->categories as $rCategory) {
                                   $category .= $book->items[0]->volumeInfo->categories[$i] . ';';
                                   $i++;
                              }
                         } else {
                              $category = $book->items[0]->volumeInfo->categories[0];
                         }
                    }

                    $countISBN = count($book->items[0]->volumeInfo->industryIdentifiers);
                    if ($countISBN > 1) {
                         foreach ($book->items[0]->volumeInfo->industryIdentifiers as $rISBN) {
                              $libISBN .= $rISBN->type . ':' . $rISBN->identifier . ';';
                         }
                    } else {
                         $libISBN = $book->items[0]->volumeInfo->industryIdentifiers[0]->type . ':' . $book->items[0]->volumeInfo->industryIdentifiers[0]->identifier;
                    }

                    $data['book'] = array(
                         'libraryTitle' => empty($book->items[0]->volumeInfo) ? '' : $book->items[0]->volumeInfo->title,
                         'librarySubTitle' => $subTitle,
                         'libraryAuthor' => empty($book->items[0]->volumeInfo) ? '' : $book->items[0]->volumeInfo->authors[0],
                         'libraryAdditionalAuthor' => $author2,
                         'libraryDescription' => empty($book->items[0]->volumeInfo->description) ? '' : trim($book->items[0]->volumeInfo->description),
                         'libraryPublisher' => empty($book->items[0]->volumeInfo->publisher) ? '' : $book->items[0]->volumeInfo->publisher,
                         'libraryYearPublished' => empty($book->items[0]->volumeInfo->publishedDate) ? '' : substr($book->items[0]->volumeInfo->publishedDate, 0, 4),
                         'libraryColation' => empty($book->items[0]->volumeInfo->pageCount) ? '' : $book->items[0]->volumeInfo->pageCount . ' halaman',
                         'libraryCategory' => $category,
                         'libraryImageLink' => empty($book->items[0]->volumeInfo->imageLinks) ? '' : $book->items[0]->volumeInfo->imageLinks->thumbnail,
                         'libraryUrl' => empty($book->items[0]->volumeInfo) ? '' : $book->items[0]->volumeInfo->infoLink,
                         'libraryISBN' => $libISBN,
                         'isbn' => $isbn,
                         'imageLink' =>  empty($book->items[0]->volumeInfo->imageLinks) ? '' : $book->items[0]->volumeInfo->imageLinks->thumbnail
                    );
               } else {
                    $data['book'] = array(
                         'libraryTitle' => '',
                         'librarySubTitle' => '',
                         'libraryAuthor' => '',
                         'libraryAdditionalAuthor' => '',
                         'libraryDescription' => '',
                         'libraryPublisher' => '',
                         'libraryYearPublished' => '',
                         'libraryColation' => '',
                         'libraryCategory' => '',
                         'libraryImageLink' => '',
                         'libraryUrl' => '',
                         'libraryISBN' => '',
                         'isbn' => '',
                         'imageLink' => ''
                    );
               }
               $this->db->where('active', 1);
               $data['bahasa'] = $this->db->get('library_language')->result_array();
               $data['fieldLang'] = array('languageID', 'languageName');
               $data['langDefault'] = $this->settingmodel->table('library_language', 'languageIsDefault ="1"')->row();

               //materi koleksi  (tesis, buku, majalah)
               $data['materi'] = $this->allmodel->getTable('material_type', ' active="1"', '', 'materialType')->result_array();
               $data['fieldMaterial'] = array('id', 'materialType');

               $data['activeView'] = 'bibliografi/v_bibliografi_add';
               }
               // echo $book->items[0]->volumeInfo->printType;
          } //end if btn cek
          else {
               $data['activeView'] = 'bibliografi/v_bibliografi_add_isbn';
               // $this->load->view('v_home',$data);
          }
          $this->load->view('v_home', $data);
     } //end if login
     else {
          $this->index();
     }
}
```
to
```php
public function create() //done
{
     $data['BASE_CTRL'] = $this->base_ctrl;
     $data['title'] = 'Tambah Bibliografi';
     $data['cari'] = '';
     $sendData = [
          'bahasa' => $this->allModel->getTable('library_language', ['active' => 1])->getResultArray(),
          'materi' => $this->allModel->getTable('material_type', ['active' => 1])->getResultArray()
     ];
     $data['dataDetail'] = ['activeView' => 'bibliografi/partials/biblio_create', 'data' => $sendData];
     return view('bibliografi/biblio', $data);
}
```
view tambahan input by pihak ke 3
```js
<script language="JavaScript" type="text/javascript">
    async function ddc(id) {
        const url = "<?= base_url() ?>ddc/" + id;
        const response = await fetch(url);
        if (response.status === 200) {
            const res = await response.json();
            if (res.msg) {
                document.getElementById("tSubyek").value = res.data;
            }
        }
    }

    document.getElementById('globalres').style.display = 'none';
    async function searchBooks() {
        var selectedAPI = document.querySelector('input[name="fapi"]:checked').value;
        var apiISBN = document.getElementById('apiISBN').value;
        var apiUrl = '';

        if (selectedAPI === 'gapis') {
            apiUrl = 'https://www.googleapis.com/books/v1/volumes?q=' + apiISBN;
        } else if (selectedAPI === 'openl') {
            apiUrl = 'https://openlibrary.org/api/books?bibkeys=ISBN:' + apiISBN + '&jscmd=details&format=json';
        }

        try {
            const response = await fetch(apiUrl);
            const data = await response.json();

            document.getElementById('globalres').style.display = 'block';
            document.getElementById('headSH').innerText = 'Hasil Referensi Sumber Lain :: ' + selectedAPI.toUpperCase() + ' :: ISBN ' + apiISBN;

            if (selectedAPI === 'gapis') {
                const volumeInfo = data.items && data.items.length > 0 ? data.items[0].volumeInfo : {};
                const industryIdentifiers = volumeInfo.industryIdentifiers || [];
                const isbn_10 = industryIdentifiers.find(identifier => identifier.type === 'ISBN_10')?.identifier || '';
                const isbn_13 = industryIdentifiers.find(identifier => identifier.type === 'ISBN_13')?.identifier || '';

                document.getElementById('result_from_api').innerHTML = `
                <span class="fw-bold fs-6 text-gray-700">Title: ${volumeInfo.title || ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Authors: ${volumeInfo.authors ? volumeInfo.authors.join(', ') : ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Publisher: ${volumeInfo.publisher || ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Published Date: ${volumeInfo.publishedDate || ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">ISBN 10: ${isbn_10}</span><br>
                <span class="fw-bold fs-6 text-gray-700">ISBN 13: ${isbn_13}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Image Links: ${volumeInfo.imageLinks ? volumeInfo.imageLinks.thumbnail || '' : ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Preview Link: ${volumeInfo.previewLink || ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Physical Format: ${volumeInfo.physical_format || ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Subjects: ${volumeInfo.subjects ? volumeInfo.subjects.join(', ') : ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Languages: ${volumeInfo.language || ''}</span><br>
            `;
            } else if (selectedAPI === 'openl') {
                const details = data['ISBN:' + apiISBN]?.details || {};

                document.getElementById('result_from_api').innerHTML = `
                <span class="fw-bold fs-6 text-gray-700">Title: ${details.title || ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Authors: ${details.authors ? details.authors.map(author => author.name).join(', ') : ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Publisher: ${details.publishers ? details.publishers[0] || '' : ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Published Date: ${details.publish_date || ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">ISBN 10: ${details.isbn_10 ? details.isbn_10[0] || '' : ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">ISBN 13: ${details.isbn_13 ? details.isbn_13[0] || '' : ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Physical Format: ${details.physical_format || ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Subjects: ${details.subjects ? details.subjects.join(', ') : ''}</span><br>
                <span class="fw-bold fs-6 text-gray-700">Languages: ${details.languages ? details.languages.map(lang => lang.key.split('/').pop()).join(', ') : ''}</span><br>
            `;
            }
        } catch (error) {
            document.getElementById('globalres').style.display = 'none';
            console.error('Error:', error);
        }
    }

    function disRes() {
        document.getElementById('globalres').style.display = 'none';
    }
</script>
```
- **Edit Bibliografi** [&#10003;]
```php
// to
public function update($libraryID) //done
{
     $resBook = $this->allModel->getTable('library', ['libraryID' => $libraryID]);
     if ($resBook->getNumRows() > 0) {
          $data['BASE_CTRL'] = $this->base_ctrl;
          $data['title'] = 'Edit Bibliografi';
          $data['cari'] = '';
          $sendData = [
               'book' => $resBook->getRowArray(),
               'bahasa' => $this->allModel->getTable('library_language', ['active' => 1])->getResultArray(),
               'materi' => $this->allModel->getTable('material_type', ['active' => 1])->getResultArray()
          ];
          $data['dataDetail'] = ['activeView' => 'bibliografi/partials/biblio_edit', 'data' => $sendData];
          return view('bibliografi/biblio', $data);
     } else {
          return redirect()->to('/show404');
     }
}
```
- **Prose Add dan Edit Bibliografi** [&#10003;]
```php
function bibliografi_post()
{
     $data['login'] = $this->auth->cekLOGIN($this->modul);
     if ($data['login']) {
          $this->load->library('form_validation');
          $this->form_validation->set_rules('tISBN', '', '');
          $this->form_validation->set_rules('tJudul', '<b>Judul</b>', 'required');
          $this->form_validation->set_rules('tJudulTambahan', 'Judul Tambahan', '');
          $this->form_validation->set_rules('tPengarang', '', '');
          $this->form_validation->set_rules('tPenerbit', '', '');
          $this->form_validation->set_rules('tKotaTerbit', '', '');
          $this->form_validation->set_rules('tTahunTerbit', '', '');
          $this->form_validation->set_rules('tCallNumber', '<b>Call Number</b>', 'required');
          $this->form_validation->set_rules('tSubyek', '', '');
          $this->form_validation->set_rules('tMateri', '<b>Materi</b>', '');
          $this->form_validation->set_rules('tBahasa', '<b>Bahasa</b>', '');
          $this->form_validation->set_rules('tLink', '', '');

          if ($this->form_validation->run() == FALSE) {
               $pesan = array('Input gagal, data harus lengkap', 'danger');
               $this->add($pesan);
          } else {
               $pathYear = './assets/files/cover/' . date('Y');
               $pathMonth = '/' . date('m');

               $allPath = $pathYear . $pathMonth . '/';

               /* Move File index to coverImages folder */
               $pathIndex = './assets/files/cover/index.html';
               $destPathIndex = $allPath . 'index.html';

               if (!is_dir($pathYear)) {
               mkdir($pathYear, 0777, TRUE); //create dir
               rename($pathIndex, $destPathIndex); //create index dir
               }

               if (!is_dir($pathYear . $pathMonth)) {
               mkdir($pathYear . $pathMonth, 0755, TRUE);
               rename($pathIndex, $destPathIndex);
               }

               // $config['upload_path']          = './assets/files/cover/'.date('Y').'/';
               $config['upload_path']          = $allPath; //NEW path
               $config['allowed_types']        = 'image/jpeg|jpg|jpeg|png';
               $config['max_size']             = 0;
               $config['max_width']            = 0;
               $config['max_height']           = 0;

               if ($this->input->post('bSimpan') == 'Save') {
               $imageLink = $this->input->post('tImageLink');
               if ($imageLink == '') {
                    $gbFilesize = $_FILES['fImage']['size'];
                    if ($gbFilesize > 0) {
                         $gbFilename = $_FILES['fImage']['name'];
                         $gbFiletype = $_FILES['fImage']['type'];
                         // $gbImage = file_get_contents($_FILES['fImage']['tmp_name']);
                    } // if >0
                    else {
                         $gbFilename = '';
                         $gbFilesize = 0;
                         $gbFiletype = '';
                         $gbImage = '';
                    }
                    $newBIB = array(
                         'libraryTitle' => $this->iPost('tJudul'),
                         'libraryDate' => date('Y-m-d H:i:s'),
                         'staffAccount' => $this->session->userdata('userID'),
                         'ownerID' => $this->session->userdata('ownerID'),
                         'libraryMaterialType' => $this->iPost('tMateri'),
                         'libraryLanguage' => $this->iPost('tBahasa'),
                         'libraryCallNumber' => $this->iPost('tCallNumber'),
                         'libraryISBN' => $this->iPost('tISBN'),
                         'libraryAuthor' => $this->iPost('tPengarang'),
                         'libraryAdditionalAuthor' => $this->iPost('tPengarangTambahan'),
                         'libraryCooperate' => $this->iPost('tKorporasi'),
                         'librarySubTitle' => $this->iPost('tJudulTambahan'),
                         'libraryEdition' => $this->iPost('tEdisi'),
                         'libraryCityPublished' => $this->iPost('tKotaTerbit'),
                         'libraryPublisher' => $this->iPost('tPenerbit'),
                         'libraryYearPublished' => $this->iPost('tTahunTerbit'),
                         'libraryVolume' => $this->iPost('tVolume'),
                         'libraryDesc' => $this->iPost('tDeskripsi'),
                         'libraryUrl' => $this->iPost('tLink'),
                         'librarySeries' => $this->iPost('tSeri'),
                         'librarySubject' => $this->iPost('tSubyek'),
                         'libraryKeyword' => $this->iPost('tKeyword'),
                         'libraryNotes' => $this->iPost('tKolasi'),
                         // 'libraryImage' => $gbImage,
                         // 'libraryImageName' => $gbFilename,
                         'libraryImageType' => $gbFiletype,
                         'libraryImageSize' => $gbFilesize
                    );
               } else {
                    $newBIB = array(
                         'libraryTitle' => $this->iPost('tJudul'),
                         'libraryDate' => date('Y-m-d H:i:s'),
                         'staffAccount' => $this->session->userdata('userID'),
                         'ownerID' => $this->session->userdata('ownerID'),
                         'libraryMaterialType' => $this->iPost('tMateri'),
                         'libraryLanguage' => $this->iPost('tBahasa'),
                         'libraryCallNumber' => $this->iPost('tCallNumber'),
                         'libraryISBN' => $this->iPost('tISBN'),
                         'libraryAuthor' => $this->iPost('tPengarang'),
                         'libraryAdditionalAuthor' => $this->iPost('tPengarangTambahan'),
                         'libraryCooperate' => $this->iPost('tKorporasi'),
                         'librarySubTitle' => $this->iPost('tJudulTambahan'),
                         'libraryEdition' => $this->iPost('tEdisi'),
                         'libraryCityPublished' => $this->iPost('tKotaTerbit'),
                         'libraryPublisher' => $this->iPost('tPenerbit'),
                         'libraryYearPublished' => $this->iPost('tTahunTerbit'),
                         'libraryVolume' => $this->iPost('tVolume'),
                         'libraryDesc' => $this->iPost('tDeskripsi'),
                         'libraryUrl' => $this->iPost('tLink'),
                         'librarySeries' => $this->iPost('tSeri'),
                         'librarySubject' => $this->iPost('tSubyek'),
                         'libraryKeyword' => $this->iPost('tKeyword'),
                         'libraryNotes' => $this->iPost('tKolasi'),
                         'libraryImageLink' => $this->input->post('tImageLink')
                    );
               }
               $this->db->insert('library', $newBIB);
               $pesan = array('Berhasil input data <B>' . $this->iPost('tJudul') . '</b><br>', 'info');
               //$this->index($pesan);
               //redirect to new Insert
               $qRedirect = 'SELECT max(libraryID) as lastInsert FROM library where staffAccount=' . $this->session->userdata('userID');
               $rRedirect = $this->db->query($qRedirect)->row();

               //upload image
               if ($gbFilesize > 0) {
                    $arrFileType = explode('.', $gbFilename);
                    $newImageName = $rRedirect->lastInsert . '.' . $arrFileType[1];
                    $config['file_name'] = $newImageName;
                    $this->load->library('upload', $config);
                    if (!$this->upload->do_upload('fImage')) {
                         $error = array('error' => $this->upload->display_errors());
                         echo 'gbFilename: ' . $gbFilename . '<br>';
                         echo 'gbFiletype: ' . $gbFiletype . '<br>';
                         echo 'New Image Name: ' . $newImageName . '<br>';
                         echo var_dump($error);
                         die;
                    } else {
                         // $gambar = array('upload_data' => $this->upload->data());
                         $this->resizeImage($config['upload_path'] . $newImageName);
                         $this->db->where('libraryID', $rRedirect->lastInsert);
                         $this->db->update('library', array('libraryImagePath' => $config['upload_path'], 'libraryImageName' => $newImageName));
                    }
               }

               redirect(base_url() . 'bibliografi/detail/' . $rRedirect->lastInsert);
               } //end if Save
               else if ($this->input->post('bSimpan') == 'Update') {
               $id = $this->input->post('libraryID');
               $tMateri = $this->input->post('tMateri');
               if ($this->input->post('tMateri') == '') {
                    $tMateri = $this->input->post('hideMateri');
               }
               $gbFilesize = $_FILES['fImage']['size'];
               if ($gbFilesize > 0) {
                    $gbFilename = $_FILES['fImage']['name'];
                    $gbFiletype = $_FILES['fImage']['type'];
                    $arrFileName = explode('.', $gbFilename);
                    // $gbImage = file_get_contents($_FILES['fImage']['tmp_name']);
                    $newEdit = array(
                         'libraryTitle' => $this->iPost('tJudul'),
                         'libraryEditDate' => date('Y-m-d H:i:s'),
                         'staffaccountEdit' => $this->session->userdata('userID'),
                         'libraryMaterialType' => $tMateri,
                         'libraryLanguage' => $this->iPost('tBahasa'),
                         'libraryCallNumber' => $this->iPost('tCallNumber'),
                         'libraryISBN' => $this->iPost('tISBN'),
                         'libraryAuthor' => $this->iPost('tPengarang'),
                         'libraryAdditionalAuthor' => $this->iPost('tPengarangTambahan'),
                         'libraryCooperate' => $this->iPost('tKorporasi'),
                         'librarySubTitle' => $this->iPost('tJudulTambahan'),
                         'libraryEdition' => $this->iPost('tEdisi'),
                         'libraryCityPublished' => $this->iPost('tKotaTerbit'),
                         'libraryPublisher' => $this->iPost('tPenerbit'),
                         'libraryYearPublished' => $this->iPost('tTahunTerbit'),
                         'libraryVolume' => $this->iPost('tVolume'),
                         'libraryDesc' => $this->iPost('tDeskripsi'),
                         'libraryUrl' => $this->iPost('tLink'),
                         'librarySeries' => $this->iPost('tSeri'),
                         'librarySubject' => $this->iPost('tSubyek'),
                         'libraryKeyword' => $this->iPost('tKeyword'),
                         'libraryNotes' => $this->iPost('tKolasi'),
                         // 'libraryImage' => '',
                         'libraryImagePath' => $config['upload_path'],
                         'libraryImageName' => $id . '.' . $arrFileName[1],
                         'libraryImageType' => $gbFiletype,
                         'libraryImageSize' => $gbFilesize
                    );
                    // upload gambar baru
                    $config['file_name'] = $id . '.' . $arrFileName[1];
                    $config['overwrite'] = true;
                    $this->load->library('upload', $config);
                    if (!$this->upload->do_upload('fImage')) {
                         $error = array('error' => $this->upload->display_errors());
                         echo 'gbFilename: ' . $gbFilename . '<br>';
                         echo 'gbFiletype: ' . $gbFiletype . '<br>';
                         echo 'New Image Name: ' . $id . '.' . $arrFileName[1] . '<br>';
                         echo var_dump($error);
                         die;
                    } else {
                         // $gambar = array('upload_data' => $this->upload->data());
                         $this->resizeImage($config['upload_path'] . $id . '.' . $arrFileName[1]);
                         echo $config['upload_path'] . $id . '.' . $arrFileName[1];
                    }
               } // if >0
               else //jika gambar ga di ubah
               {
                    $newEdit = array(
                         'libraryTitle' => $this->iPost('tJudul'),
                         'libraryEditDate' => date('Y-m-d H:i:s'),
                         'staffaccountEdit' => $this->session->userdata('userID'),
                         'libraryMaterialType' => $tMateri,
                         'libraryLanguage' => $this->iPost('tBahasa'),
                         'libraryCallNumber' => $this->iPost('tCallNumber'),
                         'libraryISBN' => $this->iPost('tISBN'),
                         'libraryAuthor' => $this->iPost('tPengarang'),
                         'libraryAdditionalAuthor' => $this->iPost('tPengarangTambahan'),
                         'libraryCooperate' => $this->iPost('tKorporasi'),
                         'librarySubTitle' => $this->iPost('tJudulTambahan'),
                         'libraryEdition' => $this->iPost('tEdisi'),
                         'libraryCityPublished' => $this->iPost('tKotaTerbit'),
                         'libraryPublisher' => $this->iPost('tPenerbit'),
                         'libraryYearPublished' => $this->iPost('tTahunTerbit'),
                         'libraryVolume' => $this->iPost('tVolume'),
                         'libraryDesc' => $this->iPost('tDeskripsi'),
                         'libraryUrl' => $this->iPost('tLink'),
                         'librarySeries' => $this->iPost('tSeri'),
                         'librarySubject' => $this->iPost('tSubyek'),
                         'libraryKeyword' => $this->iPost('tKeyword'),
                         'libraryNotes' => $this->iPost('tKolasi')
                    );
               } //end jika no edit gambar
               $lib = $this->allmodel->getTable($this->table, 'libraryID ="' . $id . '" LIMIT 1');

               //update data
               $this->db->where('libraryID', $id);
               $this->db->update('library', $newEdit);

               //catat log
               $fields = $this->db->list_fields('library');
               $beaf = '';
               $triger = $this->config->item('editTriger');
               foreach ($fields as $field) {
                    if (array_key_exists($field, $newEdit)) {
                         if ($lib->row()->$field != $newEdit[$field]) {
                              if (array_key_exists($field, $triger)) {
                                   if ($field != 'libraryImage') {
                                   $before = $this->settingmodel->table($triger[$field][0], $triger[$field][1] . ' ="' . $lib->row()->$field . '"');
                                   $after = $this->settingmodel->table($triger[$field][0], $triger[$field][1] . ' ="' . $newEdit[$field] . '"');
                                   if ($before->num_rows() > 0) {
                                        $before = $before->row()->$triger[$field][2];
                                        $before = '[' . $lib->row()->$field . ']' . $before;
                                   } else {
                                        $before = $lib->row()->$field;
                                   }
                                   if ($after->num_rows() > 0) {
                                        $after = $after->row()->$triger[$field][2];
                                        $after = '[' . $newEdit[$field] . ']' . $after;
                                   } else {
                                        $after = $newEdit[$field];
                                   }
                                   $beaf .= $field . '*' . $before . '=>' . $after . ';<br> ';
                                   } //end if !=libraryImage
                              } else {
                                   if ($field != 'libraryImage') {
                                   $beaf .= $field . '*' . $lib->row()->$field . '=>' . $newEdit[$field] . ';<br> ';
                                   }
                              }
                         }
                    }
               }
               //echo $beaf;
               if ($beaf != '') {
                    $log = array(
                         'staffID' => $this->session->userdata('userID'),
                         'libraryID' => $id,
                         'log' => $beaf,
                         'date' => date('Y-m-d H:i:s'),
                         'ownerID' => $this->ownerID
                    );
                    $this->db->insert('library_log', $log);
               }
               redirect($this->index() . '/bibliografi/detail/' . $id);
               } //end else if Update
               else {
               $this->index();
               }
          } //end form valid
     } //end if data['login']
     else {
          $this->index();
     }
}
```
to
```php
//insert
function create_proses() //done
{
     $validation = \Config\Services::validation();
     $file = $this->request->getFile('fImage');
     $validation->setRules([
          'fImage' => 'max_size[fImage,4048]|ext_in[fImage,jpeg,jpg,png]'
     ]);

     if ($validation->run() == FALSE) {
          return redirect()->back()->withInput()->with('pesan', ['danger', $validation->getError('staffImage')]);
     } else {
          $pathYear = './assets/files/cover/' . date('Y');
          $pathMonth = '/' . date('m');
          $allPath = $pathYear . $pathMonth . '/';
          $this->create_dir($pathYear, $pathMonth, $allPath);

          $newBIB = array(
               'libraryDate' => date('Y-m-d H:i:s'),
               'staffAccount' => $this->session->userID,
               'libraryTitle' => $this->iPost('tJudul'),
               'librarySubTitle' => $this->iPost('tJudulTambahan'),
               'libraryAuthor' => $this->iPost('tPengarang'),
               'libraryAdditionalAuthor' => $this->iPost('tPengarangTambahan'),
               'libraryCooperate' => $this->iPost('tKorporasi'),
               'libraryEdition' => $this->iPost('tEdisi'),
               'libraryVolume' => $this->iPost('tVolume'),
               'libraryPublisher' => $this->iPost('tPenerbit'),
               'libraryCityPublished' => $this->iPost('tKotaTerbit'),
               'libraryProduction' => $this->iPost('tProduksi'),
               'libraryDistribution' => $this->iPost('tDistribusi'),
               'libraryCreation' => $this->iPost('tPembuatan'),
               'libraryYearPublished' => $this->iPost('tTahunTerbit'),
               'librarySeries' => $this->iPost('tSeri'),
               'libraryISBN' => $this->iPost('tISBN'),
               'libraryMaterialType' => $this->iPost('tMateri'),
               'libraryNotes' => $this->iPost('tKolasi'),
               'libraryCallNumber' => $this->iPost('tCallNumber'),
               'librarySubject' => $this->iPost('tSubyek'),
               'libraryLanguage' => $this->iPost('tBahasa'),
               'libraryImageLink' => $this->iPost('tImageLink'),
               'libraryUrl' => $this->iPost('tLink'),
               'libraryKeyword' => $this->iPost('tKeyword'),
               'libraryDesc' => $this->iPost('tDeskripsi'),
               'ownerID' => $this->session->ownerID,
          );

          $lastInID = $this->allModel->addT('library', $newBIB);
          if ($lastInID != null) {
               if ($file->isValid() && !$file->hasMoved()) {
                    $ext = $file->getClientExtension();
                    $filename = $lastInID . '.' . $ext;
                    $type = $file->getClientMimeType();
                    $sizeInBytes = $file->getSize();
                    $size = intval($sizeInBytes / 1024);
                    $file->move($allPath, $filename);
                    $this->resizeImage($size, $allPath, $filename);

                    $upImage = [
                         'libraryImagePath' => $allPath,
                         'libraryImageName' => $filename,
                         'libraryImageType' => $type,
                         'libraryImageSize' => $size
                    ];
               } else {
                    $upImage = [
                         'libraryImagePath' => NULL,
                         'libraryImageName' => '',
                         'libraryImageType' => '',
                         'libraryImageSize' => 0
                    ];
               }
               $this->allModel->edit('library', ['libraryID' => $lastInID], $upImage);
               return redirect()->to($this->base_ctrl . '/detail/' . $lastInID)->with('pesan', ['success', 'Berhasil Input Data <b>' . $this->iPost('tJudul') . '</b>']);
          } else {
               return redirect()->back()->with('pesan', ['danger', '<b>Maaf, Gagal Input Bibliografi.</b>, Silahkan Coba Lagi!']);
          }
     }
}
//update
function update_proses() //done
{
     $validation = \Config\Services::validation();
     $file = $this->request->getFile('fImage');
     $validation->setRules([
          'fImage' => 'max_size[fImage,4048]|ext_in[fImage,jpeg,jpg,png]'
     ]);

     if ($validation->run() == FALSE) {
          return redirect()->back()->withInput()->with('pesan', ['danger', $validation->getError('fImage')]);
     } else {
          $id = $this->request->getPost('libraryID');
          // before data
          $beforeData = $this->allModel->getTable('library', ['libraryID' => $id]);

          if ($file->isValid() && !$file->hasMoved()) {
               $pathYear = './assets/files/cover/' . date('Y');
               $pathMonth = '/' . date('m');
               $allPath = $pathYear . $pathMonth . '/';
               $this->create_dir($pathYear, $pathMonth, $allPath);

               $ext = $file->getClientExtension();
               $filename = $id . '.' . $ext;
               $type = $file->getClientMimeType();
               $sizeInBytes = $file->getSize();
               $size = intval($sizeInBytes / 1024);
               $file->move($allPath, $filename);
               $this->resizeImage($size, $allPath, $filename);
          } else {
               //jika cover tidak di remove
               $avatar_remove = $this->request->getPost('avatar_remove');
               if ($avatar_remove == '') {
                    $old_img_path = $this->request->getPost('libraryImagePath');
                    $old_img_name = $this->request->getPost('libraryImageName');
                    $old_img_type = $this->request->getPost('libraryImageType');
                    $old_img_size = $this->request->getPost('libraryImageSize');
               } else {
                    $old_img_path = $old_img_name = $old_img_type = $old_img_size = '';
               }

               $allPath = $old_img_path;
               $filename = $old_img_name;
               $type = $old_img_type;
               $size = $old_img_size;
          }

          // after data
          $newEdit = [
               'libraryEditDate' => date('Y-m-d H:i:s'),
               'staffaccountEdit' => $this->session->userID,
               'libraryTitle' => $this->iPost('tJudul'),
               'librarySubTitle' => $this->iPost('tJudulTambahan'),
               'libraryAuthor' => $this->iPost('tPengarang'),
               'libraryAdditionalAuthor' => $this->iPost('tPengarangTambahan'),
               'libraryCooperate' => $this->iPost('tKorporasi'),
               'libraryEdition' => $this->iPost('tEdisi'),
               'libraryVolume' => $this->iPost('tVolume'),
               'libraryPublisher' => $this->iPost('tPenerbit'),
               'libraryCityPublished' => $this->iPost('tKotaTerbit'),
               'libraryProduction' => $this->iPost('tProduksi'),
               'libraryDistribution' => $this->iPost('tDistribusi'),
               'libraryCreation' => $this->iPost('tPembuatan'),
               'libraryYearPublished' => $this->iPost('tTahunTerbit'),
               'librarySeries' => $this->iPost('tSeri'),
               'libraryISBN' => $this->iPost('tISBN'),
               'libraryMaterialType' => $this->iPost('tMateri'),
               'libraryNotes' => $this->iPost('tKolasi'),
               'libraryCallNumber' => $this->iPost('tCallNumber'),
               'librarySubject' => $this->iPost('tSubyek'),
               'libraryLanguage' => $this->iPost('tBahasa'),
               'libraryImageLink' => $this->iPost('tImageLink'),
               'libraryUrl' => $this->iPost('tLink'),
               'libraryKeyword' => $this->iPost('tKeyword'),
               'libraryDesc' => $this->iPost('tDeskripsi'),
               'libraryImagePath' => $allPath,
               'libraryImageName' => $filename,
               'libraryImageType' => $type,
               'libraryImageSize' => $size
          ];
          $this->allModel->edit('library', ['libraryID' => $id], $newEdit);

          $log = $this->compareBeforeAfter($beforeData, $newEdit);
          if ($log != '') {
               $newlog = array(
                    'staffID' => $this->session->userID,
                    'libraryID' => $id,
                    'date' => date('Y-m-d H:i:s'),
                    'before' => $beforeData->getNumRows() > 0 ? json_encode($beforeData->getRowArray()) : NULL,
                    'after' => json_encode($newEdit),
                    'log' => $log,
                    'ownerID' => $this->session->ownerID
               );
               $this->allModel->add('library_log', $newlog);
          }

          return redirect()->to($this->base_ctrl . '/detail/' . $id)->with('pesan', ['success', 'Berhasil Edit Data <b>' . $this->iPost('tJudul') . '</b>']);
     }
}
```
- **Save Log Edit Bibliografi** [&#10003;]
```php
public function compareBeforeAfter($before, $after) //done
{
     $log = '';
     if ($before->getNumRows()) {
          foreach ($before->getRowArray() as $key => $valueBefore) {
               if (array_key_exists($key, $after)) {
                    $valueAfter = $after[$key];
                    if ($valueBefore !== $valueAfter) {
                         $log .= "$key * $valueBefore => $key * $valueAfter <br>";
                    }
               }
          }
     }
     return $log;
}
```
- **Detail Bibliografi** [&#10003;]
```php
function detail($pesan = '', $libID = '')
{
     $data['login'] = $this->auth->cekLOGIN($this->modul);

     if ($data['login']) {
          if ($this->uri->segment(3) != '') {
               $libraryID = $this->uri->segment(3);
          } else {
               $libraryID = $libID;
          }

          if ($libraryID != '') {
               $data['title'] = $this->lang->line('detail') . ' ' . $this->lang->line('bibliografi');
               $data['book'] = $this->bibmodel->detailBibliografi($libraryID)->row_array();
               $data['login'] = $this->auth->cekLOGIN($this->modul);
               $data['activeView'] = $this->base_view . '/v_bibliografi_detail';

               if ($pesan != '') {
               $data['pesan'] = $pesan;
               } else {
               $data['pesan'] = '';
               }
               $data['BASE_CTRL'] = $this->base_ctrl;
               $data['staffEdit'] = '';
               if ($data['book']['staffaccountEdit'] != '') {
               $data['staffEdit'] = $this->allmodel->getField('staff', 'staffID', 'staffName', $data['book']['staffaccountEdit']);
               }

               // $data['owners'] = $this->bibmodel->ownersBibliografi($libraryID, '')->result_array();

               //20221010  ga kepake
               // $userITBNumber = $this->session->userdata('userITBNumber');
               // if (($this->session->userdata('userStatus') == 'Mahasiswa') || ($this->session->userdata('userStatus') == 'Dosen')) {
               //     $whereReservation = '(owner.ownerAllowNIM="*" OR owner.ownerAllowNIM like "%' . substr($userITBNumber, 0, 3) . '%") ';
               //     $data['ownersReservation'] = $this->bibmodel->ownersBibliografi($libraryID, $whereReservation)->result_array();
               // }

               $data['joinMK'] = $this->db->get_where('course_library', array('libraryID' => $libraryID));

               $this->load->view('v_home', $data);
          }
     } else {
          redirect(base_url());
     }
}
```
to
```php
//pindah query dari view ke controller
//NOTE: 1 library 1 Owner kan ?
public function read($libraryID) //done
{
     $resBook = $this->bibmodel->detailBibliografi($libraryID);
     if ($resBook->getNumRows() > 0) {
          $data['BASE_CTRL'] = $this->base_ctrl;
          $data['title'] = 'Detail Bibliografi';
          $data['cari'] = '';
          $bookDetails = $resBook->getRowArray();
          $sendData = [
               'book' => $bookDetails,
               'eksem' => $this->bibmodel->detailEks($bookDetails['libraryID'], $bookDetails['ownerID']),
               'unit' => $this->bibmodel->detailUnit($bookDetails['libraryID'], '', $bookDetails['ownerID']),
               'onLoan' => $this->bibmodel->detailOnLoan($bookDetails['libraryID'], $bookDetails['ownerID']),
               'dataWeeding' => $this->bibmodel->detailOnWeeding($bookDetails['libraryID'], $bookDetails['ownerID']),
               'dataBookLost' => $this->bibmodel->detailOnLost($bookDetails['libraryID'], $bookDetails['ownerID']),
               'dataRepair' => $this->bibmodel->detailOnRepair($bookDetails['libraryID'], $bookDetails['ownerID']),
               'joinMK' => $this->allModel->getTable('course_library', ['libraryID' => $libraryID])
          ];
          $data['dataDetail'] = ['activeView' => 'bibliografi/partials/biblio_detail', 'data' => $sendData];
          return view('bibliografi/biblio', $data);
     } else {
          return redirect()->to('/show404');
     }
}
```
- **Tambah Data Unit dan Multiple** [&#10003;]

from
```php
function dataunit_post()
{
     $data['login'] = $this->auth->cekLOGIN($this->modul);

     if ($data['login']) {
          $mn = array();
          $pesan = '';
          date_default_timezone_set('Asia/Jakarta');

          $isRfid = 0;
          $isRfid = $this->iPost('cbRFID');

          $barcodeLama = $this->input->post('libraryMainNumberLama');

          $mn['libraryMainNumber'] = $this->iPost('tLibraryMainNumber');
          $mn['libraryID'] = $this->iPost('tLibraryID');
          $mn['ownerID'] = $this->session->userdata('ownerID');
          $mn['collectionID'] = $this->iPost('tKoleksi');
          $mn['libraryLocation'] = $this->iPost('tLokasi');
          $mn['libraryPrice'] = $this->iPost('tHarga');
          $mn['procurementID'] = $this->iPost('tPerolehan');
          $mn['libraryOrderDate'] = $this->iPost('tTglPesan');
          $mn['libraryArriveDate'] = $this->iPost('tTglTiba');
          $mn['libraryAsetCode'] = $this->iPost('tAsetCode');
          $mn['libraryDesc'] = $this->iPost('tCatatan');
          //new
          $mn['libraryRfid'] = $isRfid;
          $mn['staffRfid'] = $this->session->userdata('userID');
          $mn['tglRfid'] = date('Y-m-d H:i:s');

          if ($_POST['bSimpan'] == 'Save') {
               $mn['date'] = date('Y-m-d H:i:s');
               $mn['staffAccount'] = $this->session->userdata('userID');
               $pesan = $this->bibmodel->insertDU($mn);
          } else if ($_POST['bSimpan'] == 'Update') {
               $mn['dateUpdate'] = date('Y-m-d H:i:s');
               $mn['staffaccountEdit'] = $this->session->userdata('userID');
               $pesan = $this->bibmodel->editDU($barcodeLama, $mn, $this->session->userdata('ownerID'));
          } else if ($_POST['bSimpan'] == 'DeleteDataUnit') {
               $libraryID = $this->input->post('tLibraryID');
               $ownerID = $this->session->userdata('ownerID');
               $unitsDelete = $this->input->post('unitsDelete');

               $countUnitDelete = count(explode(',', $unitsDelete));

               if ($countUnitDelete > 1) {
               $sql = 'DELETE FROM library_dataunit WHERE libraryMainNumber IN(' . $unitsDelete . ') AND libraryID="' . $libraryID . '" AND ownerID="' . $ownerID . '"';
               } else {
               $sql = 'DELETE FROM library_dataunit WHERE libraryMainNumber="' . $unitsDelete . '" AND libraryID="' . $libraryID . '" AND ownerID="' . $ownerID . '"';
               }
               $this->db->query($sql);
               $pesan = array('Berhasil hapus data ' . $unitsDelete, 'info');
          } else {
               $pesan = array();
          }
          $this->detail($pesan, $this->input->post('tLibraryID'));
     } else {
          redirect(base_url());
     }
}

function multipleDataunit_post()
{
     $data['login'] = $this->auth->cekLOGIN($this->modul);

     if ($data['login']) {
          $mn = array();
          $pesan = '';
          date_default_timezone_set('Asia/Jakarta');
          $start = $this->iPost('tStart');
          $end = $this->iPost('tEnd');
          $prefix = $this->iPost('tPrefix');
          $isRfid = 0;
          $isRfid = $this->iPost('cbRFID');

          $barcodeLama = $this->input->post('libraryMainNumberLama');
          if ((is_numeric($start)) && (is_numeric($end))) {
               $no = 1;
               for ($i = $start; $i <= $end; $i++) {
               if ($i < 10) {
                    $bar = '0000' . $i;
               } else if ($i < 100) {
                    $bar = '000' . $i;
               } else if ($i < 1000) {
                    $bar = '00' . $i;
               } else if ($i < 10000) {
                    $bar = '0' . $i;
               } else {
                    $bar = $i;
               }
               $mn['libraryMainNumber'] = $prefix . $bar;
               $mn['libraryID'] = $this->iPost('tLibraryID');
               $mn['ownerID'] = $this->session->userdata('ownerID');
               $mn['collectionID'] = $this->iPost('tKoleksi');
               $mn['libraryLocation'] = $this->iPost('tLokasi');
               $mn['libraryPrice'] = $this->iPost('tHarga');
               $mn['procurementID'] = $this->iPost('tPerolehan');
               $mn['libraryOrderDate'] = $this->iPost('tTglPesan');
               $mn['libraryArriveDate'] = $this->iPost('tTglTiba');
               $mn['libraryAsetCode'] = $this->iPost('tAsetCode');
               $mn['libraryDesc'] = $this->iPost('tCatatan');
               //new
               $mn['libraryRfid'] = $isRfid;
               $mn['staffRfid'] = $this->session->userdata('userID');
               $mn['tglRfid'] = date('Y-m-d H:i:s');
               if ($_POST['bSimpan'] == 'Save') {
                    $mn['date'] = date('Y-m-d H:i:s');
                    $mn['staffAccount'] = $this->session->userdata('userID');
                    $pesan = $this->bibmodel->insertDU($mn);
               } else if ($_POST['bSimpan'] == 'Update') {
                    $mn['dateUpdate'] = date('Y-m-d H:i:s');
                    $mn['staffaccountEdit'] = $this->session->userdata('userID');
                    $pesan = $this->bibmodel->editDU($barcodeLama, $mn, $this->session->userdata('ownerID'));
               } else if ($_POST['bSimpan'] == 'DeleteDataUnit') {
                    $libraryID = $this->input->post('tLibraryID');
                    $ownerID = $this->session->userdata('ownerID');
                    $unitsDelete = $this->input->post('unitsDelete');

                    $countUnitDelete = count(explode(',', $unitsDelete));

                    if ($countUnitDelete > 1) {
                         $sql = 'DELETE FROM library_dataunit WHERE libraryMainNumber IN(' . $unitsDelete . ') AND libraryID="' . $libraryID . '" AND ownerID="' . $ownerID . '"';
                    } else {
                         $sql = 'DELETE FROM library_dataunit WHERE libraryMainNumber="' . $unitsDelete . '" AND libraryID="' . $libraryID . '" AND ownerID="' . $ownerID . '"';
                    }
                    $this->db->query($sql);
                    $pesan = array('Berhasil hapus data ' . $unitsDelete, 'info');
               } else {
                    $pesan = array();
               }
               } //end for
          } else {
               $pesan = array('mulai dan sampai harus ANGKA/NUMERIC', 'danger');
          }
          $this->detail($pesan, $this->input->post('tLibraryID'));
     } else {
          redirect(base_url());
     }
}
```
to
```php
// :: input data satuan
function proses_datun()
{
     $mn = array();
     $mn['libraryID'] = $this->iPost('tLibraryID');
     $mn['libraryPrice'] = $this->iPost('tHarga');
     $mn['libraryOrderDate'] = $this->iPost('tTglPesan');
     $mn['libraryArriveDate'] = $this->iPost('tTglTiba');
     $mn['libraryLocation'] = $this->iPost('tLokasi');
     $mn['libraryAsetCode'] = $this->iPost('tAsetCode');
     $mn['libraryDesc'] = $this->iPost('tCatatan');
     $mn['tglRfid'] = date('Y-m-d H:i:s');
     $mn['staffRfid'] = $this->session->userID;
     $mn['ownerID'] = $this->session->ownerID;
     $mn['collectionID'] = $this->iPost('tKoleksi');
     $mn['procurementID'] = $this->iPost('tPerolehan');
     $mn['date'] = date('Y-m-d H:i:s');
     $mn['staffAccount'] = $this->session->userID;

     //form setting
     $form = $this->request->getPost();
     if (isset($form['fsingle']) && $form['fsingle'] == 1) {
          $mn['libraryMainNumber'] = $this->iPost('tLibraryMainNumber');
          $mn['libraryRfid'] = $this->iPost('cbRFIDs') != NULL ? 1 : 0;
          $res = $this->single_datun($mn);
     } elseif (isset($form['fmulti']) && $form['fmulti'] == 1) {
          $mn['libraryRfid'] = $this->iPost('cbRFIDm') != NULL ? 1 : 0;
          $prefix = $this->iPost('tPrefix');
          $start = $this->iPost('tStart');
          $end = $this->iPost('tEnd');
          $res = $this->multi_datun($mn, $prefix, $start, $end);
     } else {
          $res = [];
     }

     if (!empty($res)) {
          return redirect()->to($this->base_ctrl . '/detail/' . $this->iPost('tLibraryID'))->with('pesan', $res);
     } else {
          return redirect()->back()->with('pesan', ['danger', 'Tidak bisa input Data Satuan, mohon coba lagi!']);
     }
}

function single_datun($mn) //done
{
     $psn = [];
     $btn = $this->request->getPost('bSimpan');
     if ($btn == 'Save') {
          $psn = $this->bibmodel->addDatun($mn);
     }
     return $psn;
}

function multi_datun($mn, $prefix, $start, $end) //done
{
     $isSukses = 0;
     $isNotSukses = 0;
     for ($i = $start; $i <= $end; $i++) {
          if ($i < 10) {
               $bar = '0000' . $i;
          } else if ($i < 100) {
               $bar = '000' . $i;
          } else if ($i < 1000) {
               $bar = '00' . $i;
          } else if ($i < 10000) {
               $bar = '0' . $i;
          } else {
               $bar = $i;
          }

          $mn['libraryMainNumber'] = $prefix . $bar;
          $status = $this->bibmodel->addMultiDatun($mn);
          if ($status) {
               $isSukses += 1;
          } else {
               $isNotSukses += 1;
          }
     }
     $psn = ['info', 'Input Multiple Data Satuan : <b>BERHASIL ' . $isSukses . ' Data Unit</b>. GAGAL ' . $isNotSukses];
     return $psn;
}
//model di pisah untuk multiple upload
function addMultiDatun($data)
{
     //banyak data pake transaksi
     $this->db->transStart();

     $existingData = $this->db->table($this->table_ld)
          ->where('ownerID', $data['ownerID'])
          ->where('libraryMainNumber', $data['libraryMainNumber'])
          ->get()
          ->getNumRows();

     if ($existingData > 0) {
          $this->db->transComplete();
          return false;
     }

     $this->db->table($this->table_ld)->insert($data);

     $libraryOwnerExists = $this->db->table($this->table_lo)
          ->where('libraryID', $data['libraryID'])
          ->where('ownerID', $data['ownerID'])
          ->get()
          ->getNumRows();

     if ($libraryOwnerExists == 0) {
          $newLibOwner = [
               'libraryID' => $data['libraryID'],
               'ownerID' => $data['ownerID']
          ];
          $this->db->table($this->table_lo)->insert($newLibOwner);
     }

     $this->db->transComplete();

     return $this->db->transStatus() !== FALSE;
}
```
- **Edit Data Unit** [&#10003;]

from
```php
function editDataUnit()
{
     $data['login'] = $this->auth->cekLOGIN($this->modul);
     if ($data['login']) {
          //detail
          $libraryID = $this->uri->segment(3);
          $mainNumber = urldecode(urldecode($this->uri->segment(4)));
          $data['book'] = $this->bibmodel->detailBibliografi($libraryID)->row_array();
          $unit = $this->bibmodel->dataUnit($libraryID, $mainNumber, $this->session->userdata('ownerID'));

          if ($unit->num_rows() > 0) {
               // $data['unit'] = $this->bibmodel->dataUnit($libraryID,$mainNumber,$this->session->userdata('ownerID'))->row_array();
               $data['unit'] = $unit->row_array();
               $data['libraryMainNumber'] = $mainNumber;
               $data['libraryID'] = $libraryID;

               $data['staffEdit'] = '';
               if ($data['book']['staffaccountEdit'] != '') {
               $data['staffEdit'] = $this->allmodel->getField('staff', 'staffID', 'staffName', $data['book']['staffaccountEdit']);
               }

               $data['owners'] = $this->bibmodel->ownersBibliografi($libraryID)->result_array();

               $data['activeView'] = $this->base_view . '/v_bibliografi_dataunit_edit';
               $data['title'] = $this->lang->line('edit') . ' ' . $this->lang->line('barcode');
               $data['BASE_CTRL'] = $this->base_ctrl;

               $this->load->view('v_home', $data);
          } else {
               redirect(base_url());
          }
     }
}
```
to
```php
// controller langsung proses
function edit_datun() //done
{
     $libID = $this->request->getPost('eLibraryID');
     $barcodeLama = $this->request->getPost('libraryMainNumberLama');
     $psn = [];
     if (!empty($libID) && !empty($barcodeLama)) {
          $mn = array();
          $mn['libraryMainNumber'] = $this->iPost('eLibraryMainNumber');
          $mn['libraryPrice'] = $this->iPost('eHarga');
          $mn['libraryOrderDate'] = $this->iPost('eTglPesan');
          $mn['libraryArriveDate'] = $this->iPost('eTglTiba');
          $mn['libraryLocation'] = $this->iPost('eLokasi');
          $mn['libraryAsetCode'] = $this->iPost('eAsetCode');
          $mn['libraryDesc'] = $this->iPost('eCatatan');
          $mn['libraryRfid'] = $this->iPost('eRFID') != NULL ? 1 : 0;
          $mn['tglRfid'] = date('Y-m-d H:i:s');
          $mn['staffRfid'] = $this->session->userID;
          $mn['ownerID'] = $this->session->ownerID;
          $mn['collectionID'] = $this->iPost('eKoleksi');
          $mn['procurementID'] = $this->iPost('ePerolehan');
          $mn['dateUpdate'] = date('Y-m-d H:i:s');
          $mn['staffaccountEdit'] = $this->session->userID;
          $psn = $this->bibmodel->editDatun($barcodeLama, $mn, $this->session->ownerID);
     }

     if (!empty($psn)) {
          return redirect()->to($this->base_ctrl . '/detail/' . $libID)->with('pesan', $psn);
     } else {
          return redirect()->back()->with('pesan', ['danger', 'Tidak bisa edit Data Satuan, mohon coba lagi!']);
     }
}
```
- **Hapus Data Unit** [&#10003;]

from
```php
function deleteDataUnitConfirm()
{
     $data['login'] = $this->auth->cekLOGIN($this->modul);
     if ($data['login']) {
          $units = $this->input->post('cbBarcode');
          $ada = empty($units);
          if (!$ada) {
               $sql = 'SELECT library.* from library join library_dataunit on library.libraryID=library_dataunit.libraryID where library_dataunit.libraryMainNumber="' . $units[0] . '"';
               $data['book'] = $this->db->query($sql)->row_array();
               $data['barcodeDelete'] = $units;
               $data['pesan'] = '';
               $data['title'] = $this->lang->line('delete') . ' ' . $this->lang->line('bibliografi');
               $data['activeView'] = $this->base_view . '/v_bibliografi_dataunit_delete_confirm';
               $data['BASE_CTRL'] = $this->base_ctrl;
               $this->load->view('v_home', $data);
          } else {
               redirect('bibliografi');
          }
     } else {
          redirect(base_url());
     }
}
```
to
```php
//controller langsung proses
function delete_datun() //done
{
     $libraryID = $this->request->getPost('dLibraryID');
     $ownerID = $this->session->ownerID;
     $unitsDelete = $this->request->getPost('cbDel');
     $countUnitDelete = count($unitsDelete);

     if (!empty($libraryID) && is_array($unitsDelete) && count($unitsDelete) > 0) {
          $countUnitDelete = count($unitsDelete);

          if ($countUnitDelete > 1) {
               $ps = implode(', ', $unitsDelete);
               $rsq = $this->allModel->deleteTransIn('library_dataunit', 'libraryMainNumber', ['libraryID' => $libraryID, 'ownerID' => $ownerID], $unitsDelete);
          } else {
               $ps = $unitsDelete[0];
               $rsq = $this->allModel->deleteTrans('library_dataunit', ['libraryMainNumber' => $unitsDelete[0], 'libraryID' => $libraryID, 'ownerID' => $ownerID]);
          }

          if ($rsq) {
               return redirect()->to($this->base_ctrl . '/detail/' . $libraryID)->with('pesan', ['warning', 'Berhasil Hapus <b>Data Satuan ' . $ps . '</b>']);
          }
     }

     return redirect()->back()->with('pesan', ['danger', 'Tidak bisa hapus Data Satuan, mohon coba lagi!']);
}
```
- **Improve View Detail Library :: porses tambah, edit, delete Data Satuan** [&#10003;]

to
```js
<script language="JavaScript" type="text/javascript">
    $("#tTglPesan").flatpickr({
        dateFormat: "Y-m-d",
    });
    $("#tTglTiba").flatpickr({
        dateFormat: "Y-m-d",
    });
    $("#eTglPesan").flatpickr({
        dateFormat: "Y-m-d",
    });
    $("#eTglTiba").flatpickr({
        dateFormat: "Y-m-d",
    });

    // Reset input fields ketika modal ditutup
    $('#kt_modal_new_target').on('hidden.bs.modal', function() {
        $(this).find('input').val('');
    });
    $('#m_edit_datun').on('hidden.bs.modal', function() {
        $(this).find('input').val('');
    });

    const singleRadio = document.getElementById('fsingle');
    const multiRadio = document.getElementById('fmulti');
    const singleInput = document.getElementById('singleinput');
    const multiInput = document.getElementById('multiInput');
    const singleUnitInput = document.getElementById('tLibraryMainNumber');
    const multipleUnitInput = document.getElementById('tPrefix');
    const tStart = document.getElementById('tStart');
    const tEnd = document.getElementById('tEnd');

    // disable tab jika salah satu input pada tab aktif terisi, biar ga bisa bolak balik
    singleUnitInput.addEventListener('input', function() {
        multiRadio.disabled = this.value.length > 0;
    });
    multipleUnitInput.addEventListener('input', function() {
        singleRadio.disabled = this.value.length > 0;
    });

    // show form jika tab di klik
    function showSingleInput() {
        singleInput.style.display = 'block';
        multiInput.style.display = 'none';
        multiRadio.checked = false;
        singleRadio.checked = true;

        singleUnitInput.setAttribute('required', 'required');
        multipleUnitInput.removeAttribute('required');
        tStart.removeAttribute('required');
        tEnd.removeAttribute('required');
    }

    function showMultiInput() {
        singleInput.style.display = 'none';
        multiInput.style.display = 'block';
        singleRadio.checked = false;
        multiRadio.checked = true;

        singleUnitInput.removeAttribute('required');
        multipleUnitInput.setAttribute('required', 'required');
        tStart.setAttribute('required', 'required');
        tEnd.setAttribute('required', 'required');
    }

    showSingleInput();

    singleRadio.addEventListener('change', function() {
        if (this.checked) {
            showSingleInput();
        }
    });

    multiRadio.addEventListener('change', function() {
        if (this.checked) {
            showMultiInput();
        }
    });

    async function editDatun(res) {
        const event = new Event('change');
        const libID = res.id;
        const libMainNumber = res.name;
        const url = "<?= base_url() ?>getDataUnit/" + libID + "/" + libMainNumber;
        const response = await fetch(url);
        if (response.status === 200) {
            const res = await response.json();
            if (res.msg) {
                const dt = res.data;

                const cb = document.getElementById('eRFID');
                if (dt.libraryRfid === '1') {
                    cb.setAttribute('checked', 'checked');
                } else {
                    cb.removeAttribute('checked');
                }

                const eKoleksi = document.getElementById('eKoleksi');
                for (let i = 0; i < eKoleksi.options.length; i++) {
                    if (eKoleksi.options[i].value == dt.collectionID) {
                        eKoleksi.selectedIndex = i;
                        eKoleksi.dispatchEvent(event);
                        break;
                    }
                }

                const ePerolehan = document.getElementById('ePerolehan');
                for (let ii = 0; ii < ePerolehan.options.length; ii++) {
                    if (ePerolehan.options[ii].value === dt.procurementID) {
                        ePerolehan.selectedIndex = ii;
                        ePerolehan.dispatchEvent(event);
                        break;
                    }
                }

                document.getElementById('eTglPesan').value = dt.libraryOrderDate;
                document.getElementById('eTglTiba').value = dt.libraryArriveDate;
                document.getElementById("libraryMainNumberLama").value = dt.libraryMainNumber;
                document.getElementById("eLibraryMainNumber").value = dt.libraryMainNumber;
                document.getElementById("eLokasi").value = dt.libraryLocation;
                document.getElementById("eAsetCode").value = dt.libraryAsetCode;
                document.getElementById("eHarga").value = dt.libraryPrice;
                document.getElementById("eCatatan").value = dt.libraryDesc;
                $('#m_edit_datun').modal("show");
            }
        } else {
            alert('Maaf! Ada kesalahan server, silahkan coba lagi nanti.');
        }
    }

    $(document).ready(function() {
        var myBtn = document.getElementById('openModalDelBtn');
        var cbx = document.querySelectorAll('input[name="cbDel[]"]');
        cbx.forEach(function(checkbox) {
            checkbox.addEventListener('change', function() {
                myBtn.disabled = !Array.from(cbx).some(function(cb) {
                    return cb.checked;
                });
            });
        });

        $('#openModalDelBtn').on('click', function() {
            var selectedBarcodes = [];
            $('input[name="cbDel[]"]:checked').each(function() {
                selectedBarcodes.push($(this).val());
            });
            $('#selectedBarcodesList').empty();
            selectedBarcodes.forEach(function(barcode) {
                $('#selectedBarcodesList').append('<li>' + barcode + '</li>');
            });
            $('#kt_modal_del_datun').modal('show');
        });

        $('#fDelDatun').on('submit', function(e) {
            e.preventDefault();
            this.submit();
        });
    });
</script>
```

- **Tambah Matakuliah**
- **Cek HPS**

```php
function create_dir($pathYear, $pathMonth, $allPath) //done not testing
{
     $pathIndex = './assets/files/cover/index.html'; /* Move File index to coverImages folder */
     $destPathIndex = $allPath . 'index.html';
     if (!is_dir($pathYear)) {
          mkdir($pathYear, 0777, TRUE); //create dir
          rename($pathIndex, $destPathIndex); //create index dir
     }
     if (!is_dir($pathYear . $pathMonth)) {
          mkdir($pathYear . $pathMonth, 0755, TRUE);
          rename($pathIndex, $destPathIndex);
     }
}
```
```js
async function xsearchBooks() {
     var selectedAPI = document.querySelector('input[name="fapi"]:checked').value;
     var apiISBN = document.getElementById('apiISBN').value;
     var apiUrl = '';

     if (selectedAPI === 'gapis') {
          apiUrl = 'https://www.googleapis.com/books/v1/volumes?q=' + apiISBN;
     } else if (selectedAPI === 'openl') {
          apiUrl = 'https://openlibrary.org/api/books?bibkeys=ISBN:' + apiISBN + '&jscmd=details&format=json';
     }

     await fetch(apiUrl)
          .then(response => response.json())
          .then(data => {
               document.getElementById('globalres').style.display = 'block';
               document.getElementById('headSH').innerText = 'Hasil Referensi Sumber Lain :: ' + selectedAPI.toUpperCase() + ' :: ' + apiISBN;
               document.getElementById('result_from_api').innerText = JSON.stringify(data, null, 9);
          })
          .catch(error => {
               document.getElementById('globalres').style.display = 'none';
               console.error('Error:', error);
          });
}
```
