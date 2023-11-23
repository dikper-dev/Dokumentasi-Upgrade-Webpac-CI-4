# Anggota/Member [&#10003;]
## Modul
- **Search** [&#10003;]
```php
//controller seacrh
$data['cari'] = $cari;
$this->db->select('u.userAccount,u.userName,u.userEmail,u.userEmailITB,u.userAddress, u.userINA, u.userPhone, u.campusCode,u.userType,u.rootID, u.officeAccount,u.userOrientationed,t.typeName, t.typeCss');
$this->db->join('user_type t', 't.id = u.userType', 'left');
$this->db->where($where);
$this->db->group_by('u.userAccount');
$cariUser = $this->db->get('user_account u');

if ($cariUser->num_rows() == 0) {
     $where = array(
          'u.userAccount' => $cari,
          'u.userINA' => $cari,
          'u.userName' => $cari
     );
     $this->db->select('u.userAccount,u.userName,u.userEmail,u.userEmailITB,u.userAddress, u.userINA, u.userPhone, u.campusCode,u.userType,u.rootID, u.officeAccount,u.userOrientationed,t.typeName, t.typeCss');
     $this->db->join('user_type t', 't.id = u.userType', 'left');
     $this->db->or_like($where, 'match');
     $this->db->group_by('u.userAccount');
     $cariUser = $this->db->get('user_account u');
}
```
to *handle query looping jika active view*
```php
// controller
if ($this->session->has('cachedUser')) {
     $cachedData = $this->session->get('cachedUser');
     $oldSearchKeyword = $cachedData['keyword'];
     if ($oldSearchKeyword === $cari) {
          $cariUser = $cachedData['data'];
     } else {
          $this->session->remove('cachedUser');
     }
}
if (!isset($cariUser)) {
 	$where = array('userAccount' => $cari);
     $cariUser = $this->membermodel->cariAnggota($where);
     if ($cariUser->getNumRows() == 0) {
          $where = array(
               'u.userAccount' => $cari,
               'u.userINA' => $cari,
               'u.userName' => $cari
          );
          $cariUser = $this->membermodel->matchAnggota($where);
    }
     $this->session->set('cachedUser', ['keyword' => $cari, 'data' => $cariUser]);
}
```
- **View Add**
```php
//view
if ($inputINA != '' && $cekUser != null) {
     $user = $cekUser->row();
     $this->atfungsi->openBlock();
     $this->atfungsi->openTable();
     echo '<tr><td>Nama</td><td>' . $user->userName . '</td></tr>';
     echo '<tr><td>NIP/Nopeg</td><td>' . $user->userAccount . '</td></tr>';
     $this->atfungsi->closeTable();
     echo '<a href="' . $BASE_CTRL . '/search?tcari=' . $user->userAccount . '" class="btn btn-info"><i class="feather icon-external-link"></i> Lihat Detil</a>';
     $this->atfungsi->closeBlock();
}
//controller
$cekUserDB = $this->db->get_where('user_account', $where);
if ($cekUserDB->num_rows()) {
     $data['pesan'] = array($itbAkun . ' sudah ada!', 'warning');
     $data['inputINA'] = $itbAkun;
     $data['cekUser'] = $cekUserDB;
} else {
     $data['user'] = array(
     'itbAkun' => $itbAkun,
     'userAccount' => $nip,
     'userINA' => $uid,
     'userName' => $cn,
     'userType' => $userType,
     'userEmailITB' => $mail,
     'userEmail' => implode(', ', explode(';', $mailnonitb)),
     'unitKerja' => $unit_kerja
     );
     $data['activeView'] = $this->base_view . '/v_member_new_add';
}
```
to *pindah controller*
```php
//langsung redirect search
$cekUserDB = $this->allModel->getTable('user_account', $where);
if ($cekUserDB->getNumRows()) {
     $cekUser = $cekUserDB->getRow();
     return redirect()->to(base_url() . 'pengguna/anggota/search?q=' . $cekUser->userAccount)->with('pesan', ['warning', $itbAkun . ' <b>Sudah ada!</b>']);
} else {
     $data['userType'] = $this->allModel->getT('user_type')->getResultArray();
     $data['user'] = array(
          'itbAkun' => $mail,
          'userAccount' => $nip,
          'userINA' => $uid,
          'userName' => $cn,
          'userType' => $userType,
          'userEmailITB' => $mail,
          'userEmail' => implode(', ', explode(';', $mailnonitb)),
          'unitKerja' => $unit_kerja
     );
}
$data['inputINA'] = $itbAkun;
return view('anggota/anggota_new_add', $data);
```
- **View Edit** [&#10003;]
     - Update Loan jika edit [&#10003;]
```php
//controller edit_post
$pesan = $this->membermodel->upData($id, $nimLama, $upData);
//remove dlu cache search soalnya search?q=userakun
$this->session->remove('cachedUser');
return redirect()->to(base_url() . 'pengguna/anggota/search?q=' . $userAccount)->with('pesan', $pesan);
```
- **Sedang Dipinjaman** [&#10003;]
     - Merubah Model
```php
//model 
function onLoanLibraryCount($userAccount)
{
     $sql = 'SELECT loan.ownerID,count(loan.ownerID) as countOwner, owner.ownerName, owner.ownerName2  
          FROM `loan` 
          JOIN owner ON owner.ownerID = loan.ownerID
          WHERE loan.userAccount = "' . $userAccount . '" AND loan.returnedDate is null
          group BY loan.ownerID ';
     $query = $this->db->query($sql);
     return $query;
} 

function onLoanMember($ownerID, $userAccount)
{
     $s = "SELECT library.libraryTitle,library.libraryID,loan.loanID, loan.libraryMainNumber,loan.returnDate,loan.loanDate,TO_DAYS(CURDATE())-TO_DAYS(loan.returnDate) as hari " .
          "FROM library " .
          "JOIN library_dataunit ON library_dataunit.libraryID=library.libraryID " .
          "JOIN loan ON loan.libraryMainNumber=library_dataunit.libraryMainNumber " .
          // "JOIN owner ON owner.ownerID=loan.ownerID " . //update arif
          "WHERE loan.ownerID = '" . $ownerID . "' AND loan.userAccount='" . $userAccount . "' AND loan.returnedDate is null " .
          "GROUP BY loan.loanID ";
     $query = $this->db->query($s);

     return $query;
}

//controller
if ($detail == 'on_loan') {
     $dataDetail = $this->membermodel->onLoanLibraryCount($userAccount);
     $viewDetail = $this->base_view . '/v_member_new_detail_onloan';
}

//view
if ($dataDetail->num_rows() > 0) { //jika masih ada pinjaman
     $onLib = $dataDetail->result_array();
     foreach ($onLib as $rOwner) {
          $this->atfungsi->openTable();
          $head = array('No','Barcode','Judul','Pinjam','Harus Kembali');
          echo $this->atfungsi->headTable($head);
          $onLoan = $this->membermodel->onLoanMember($rOwner['ownerID'], $user->userAccount)->result_array();
          $j = 1;
          foreach ($onLoan as $rLoan) {
          
          }//end foreach onloan
     }//end foreach owner
} //end jika ada pinjaman
else{
     echo 'Tidak ada Pinjaman Koleksi';
}
```
to
```php
//model gabung 2 query onLoanLibraryCount onLoanMember
function onLoanMemberLibraryCount($userAccount)
{
     $s = "SELECT 
               loan.ownerID,
               COUNT(loan.ownerID) as countOwner,
               owner.ownerName,
               owner.ownerName2,
               MAX(library.libraryTitle) as libraryTitle,
               MAX(library.libraryID) as libraryID,
               loan.loanID,
               loan.libraryMainNumber,
               loan.returnDate,
               loan.loanDate,
               TO_DAYS(CURDATE()) - TO_DAYS(loan.returnDate) as hari " .
          "FROM loan " .
          "JOIN owner ON owner.ownerID = loan.ownerID " .
          "JOIN library_dataunit ON library_dataunit.libraryMainNumber = loan.libraryMainNumber " .
          "JOIN library ON library_dataunit.libraryID = library.libraryID " .
          "WHERE loan.userAccount='" . $userAccount . "' AND loan.returnedDate is null " .
          "GROUP BY loan.loanID, loan.ownerID " .
          "ORDER BY loan.ownerID ASC";
     $query = $this->db->query($s);

     return $query;
}

//controller
if ($detail == 'on-loan') {
     $resonloan = $this->membermodel->onLoanMemberLibraryCount($userAccount);
     $dataDetail = ['activeView' => 'anggota/widgets/detail_onloan', 'data' => ['dataDetail' => $resonloan]];
}

//view proses data model
if ($dataDetail->getNumRows() > 0) :
     $headTable = ['No' => 'min-w-10px', 'Barcode' => 'min-w-100px', 'Judul' => 'min-w-200px', 'Pinjam' => 'min-w-100px', 'Harus Kembali' => 'min-w-100px'  ];

     $detailLoanPerOwner = [];
     foreach ($dataDetail->getResultArray() as $rData) {
          $ownerID = $rData['ownerID'];
          if (!isset($detailLoanPerOwner[$ownerID]['countOwner'])) {
               $detailLoanPerOwner[$ownerID]['countOwner'] = 0;
          }
          $detailLoanPerOwner[$ownerID]['countOwner']++;
          $detailLoanPerOwner[$ownerID]['ownerName'] = $rData['ownerName'];
          $detailLoanPerOwner[$ownerID]['loans'][] = [
               'ownerID' => $rData['ownerID'],
               'loanID' => $rData['loanID'],
               'ownerName' => $rData['ownerName'],
               'libraryID' => $rData['libraryID'],
               'libraryTitle' => $rData['libraryTitle'],
               'libraryMainNumber' => $rData['libraryMainNumber'],
               'loanDate' => $rData['loanDate'],
               'returnDate' => $rData['returnDate'],
               'hari' => $rData['hari'],
          ];
     }

     // var_dump($detailLoanPerOwner);
     foreach ($detailLoanPerOwner as $ownerData) {
          echo alertNotice('<i class="bi bi-building text-dark fs-2 me-2"></i>' . $ownerData['ownerName'] . ' &#8660; <b>' . $ownerData['countOwner'] . ' Pinjaman</b>', 'success');
          echo beginTableHead($headTable, 'onLoanMember');
          $no = 1;
          foreach ($ownerData['loans'] as $loan) {
               $link = base_url() . 'bibliografi/detail/' . $loan['libraryID'];
               $linkbuku = '<a href="' . $link . '" class="text-dark fw-bolder text-hover-primary d-block mb-1 fs-7">' . $loan['libraryTitle'] . '</a>';
               echo '<tr>';
               echo '<td>' . $no++ . '</td>';
               echo '<td>' . $loan['libraryMainNumber'] . '</td>';
               echo '<td>' . $linkbuku . '</td>';
               echo '<td>' . date_to_tanggal($loan['loanDate']) . '</td>';
               echo '<td>' . date_to_tanggal($loan['returnDate']) . '</td>';
               echo '</tr>';
          }
          echo endTableHead();
     }
else :
     <span class="fw-bolder fs-6 text-gray-700">Tidak ada peminjaman</span>
endif;
```
- Riwayat Peminjaman [&#10003;]
```php
//model
function loanHistory($userAccount, $groupBy)
{
     $s = "SELECT library.libraryTitle,library.libraryID,TO_DAYS(CURDATE())-TO_DAYS(loan.returnDate) as hari,
          loan.libraryMainNumber,loan.returnDate,loan.loanDate,loan.returnedDate, loan.returnedTime, loan.staffReturn, 
          owner.ownerID, owner.ownerName, staff.staffName " .
          "FROM library " .
          "JOIN library_dataunit ON library_dataunit.libraryID=library.libraryID " .
          "JOIN loan ON loan.libraryMainNumber=library_dataunit.libraryMainNumber " .
          "JOIN staff ON staff.staffID = loan.staffLoan " .
          "JOIN owner ON owner.ownerID = loan.ownerID " .
          "WHERE loan.userAccount='" . $userAccount . "' ";
     if ($groupBy != '') {
          $s .= "GROUP BY " . $groupBy;
     }
     $s .= " ORDER BY loan.loanID DESC ";
     $query = $this->db->query($s);

     return $query;
}

//controller
$dataDetail = $this->membermodel->loanHistory($userAccount, 'loan.loanID');
$viewDetail = $this->base_view . '/v_member_new_detail_history_loan';

if ($dataDetail->num_rows()) {
     $arrStaffRetur = array_column($dataDetail->result_array(), 'staffReturn');
     $this->db->where_in('staffID', $arrStaffRetur);
     $staffRetur = $this->db->get('staff');
     $data['staffReturn'] = array_column($staffRetur->result_array(), 'staffName', 'staffID');
}

//view
$statusKembali = '';
$jamKembali = '';
if (($rHL['returnedDate'] == '0000-00-00') || (is_null($rHL['returnedDate']))) {
     $statusKembali = $this->atfungsi->spanStyle('belum kembali','danger');
}
else{
     $statusKembali = $this->fungsi->date_to_tanggal($rHL['returnedDate']);
     $jamKembali = $rHL['returnedTime'].' &raquo '.$staffReturn[$rHL['staffReturn']];
}
```
to
```php
//model
function loanHistory($userAccount, $page, $perPage)
{
     $startFrom = ($page - 1) * $perPage;
     $sql = "SELECT SQL_CALC_FOUND_ROWS
               MAX(library.libraryTitle) as libraryTitle,
               MAX(library.libraryID) as libraryID,
               TO_DAYS(CURDATE())-TO_DAYS(loan.returnDate) as hari,
               loan.libraryMainNumber,
               loan.returnDate,
               loan.loanDate,
               loan.returnedDate,
               loan.returnedTime, 
               owner.ownerID,
               owner.ownerName,
               loan.staffReturn AS staffReturn,
               loan.staffLoan AS staffLoan,
               loanStaff.staffName AS staffNameLoan,
               returnStaff.staffName AS staffNameReturn
          FROM library
          JOIN library_dataunit ON library_dataunit.libraryID = library.libraryID
          JOIN loan ON loan.libraryMainNumber = library_dataunit.libraryMainNumber
          JOIN staff AS loanStaff ON loanStaff.staffID = loan.staffLoan
          JOIN owner ON owner.ownerID = loan.ownerID
          LEFT JOIN staff AS returnStaff ON returnStaff.staffID = loan.staffReturn
          WHERE loan.userAccount = '" . $userAccount . "'
          GROUP BY loan.loanID
          ORDER BY loan.loanID DESC
          LIMIT " . $startFrom . ", " . $perPage . "";
     $query = $this->db->query($sql);
     $totalRowsQuery = $this->db->query("SELECT FOUND_ROWS() as totalRows");
     $totalRows = $totalRowsQuery->getRow()->totalRows;

     return [
          'results' => $query,
          'totalRows' => $totalRows
     ]; // Return paginated data with totalRows
}

//controller
$page = (int) ($this->request->getGet('page') ?? 1);
$resloanhostory = $this->membermodel->loanHistory($userAccount, $page, $this->perPage);
$pager_links = $this->pager->makeLinks($page, $this->perPage, $resloanhostory['totalRows'], 'temp_pager');
$sendData = ['dataDetail' => $resloanhostory['results'], 'perPage' => $this->perPage, 'total' => $resloanhostory['totalRows'], 'pager_links' => $pager_links];
$dataDetail = ['activeView' => 'anggota/widgets/history_loan', 'data' => $sendData];

//view
$statusKembali = '';
$jamKembali = '';
if (($rHL['returnedDate'] == '0000-00-00') || (is_null($rHL['returnedDate']))) {
     $statusKembali = badgeStyle('danger', 'belum kembali');
} else {
     $statusKembali = date_to_tanggal($rHL['returnedDate']);
     $jamKembali = $rHL['returnedTime'] . ' &raquo ' . $rHL['staffNameReturn'];
}
```
- Pinjaman TPB [&#10003;]
- Data Kasus, Input Kasus, Tutup Kasus, Update Loan, Update FRS Wisuda json [&#10003;]
- Wisuda [&#10003;]
- FRS [&#10003;]
- SIX [&#10003;]
