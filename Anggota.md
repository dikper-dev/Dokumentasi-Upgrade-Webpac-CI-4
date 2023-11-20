# Anggota (Member)
- **Search** [&#10003;]
	- Gabung model
	```php
	//controller search
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
	to
	```php
	// new controller
	$whereMatch = array('u.userAccount' => $cari, 'u.userINA' => $cari, 'u.userName' => $cari);
	$cariUser = $this->membermodel->searchMatchAnggota($where, $whereMatch);

	//model
	function searchMatchAnggota($where, $whereMacth)
    {
        $b = $this->db->table('user_account u');
        $b->select('u.userAccount,u.userName,u.itbAccount,u.userEmail,u.userEmailITB,u.userAddress, u.userINA, u.userPhone, u.campusCode,u.userType,u.rootID, u.officeAccount,u.userOrientationed,t.typeName, t.typeCss');
        $b->join('user_type t', 't.id = u.userType', 'left');
        $b->where($where);
        $b->orLike($whereMacth, 'match');
        $b->groupBy('u.userAccount');
        return $b->get();
    }
	```
- **View Add**
- **View Edit**
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

	//view
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

	//controller
	if ($detail == 'on-loan') {
		$resonloan = $this->membermodel->onLoanMemberLibraryCount($userAccount);
		$dataDetail = ['activeView' => 'anggota/widgets/detail_onloan', 'data' => ['dataDetail' => $resonloan]];
	}
	```

- Riwayat Peminjaman [&#10003;]
- Pinjaman TPB
- Data Kasus
- Wisuda
- FRS
- SIX
