# Peminjaman (Loan)
- View (&#10003;)
- Index, Proses, Detail (&#10003;)
- **Reloan**
  *Catatan*
	- insert fine_log jika ada denda (&#10003;)
	- pengembalian ill :: belum di testing
	- update FRS (&#10003;)
	- Mengganti retur model
```php
//controller
$perpanjang = $this->returmodel->retur($save, $mainNumber, true);

//model
function retur($data, $id, $reloan=false) {
	date_default_timezone_set('Asia/Jakarta');
	$this->db->select('*, TO_DAYS(CURDATE())-TO_DAYS(loan.returnDate) as hari');
	$this->db->from('loan');
	$this->db->where('libraryMainNumber', $id);
	$this->db->where('ownerID', $data['ownerID']);
	$this->db->where('returnedDate is null');
	$this->db->order_by('loanID', 'desc');
	$this->db->limit(1);

	$loan = $this->db->get();

	if ($loan->num_rows() > 0) {
		//insert denda
		$jumlahDenda = 0;
		$adaDenda = false;
		$loanID = $loan->row();
		if ($loanID->hari > 0) {

			$minggu = $this->fungsi->getMinggu($data['ownerID'], substr($loanID->returnDate, 0, 10), date("Y-m-d"));
			$libur = $this->fungsi->getLibur($data['ownerID'], substr($loanID->returnDate, 0, 10), date("Y-m-d"));

			$lamaDenda = $loanID->hari - $minggu - $libur;
			if($lamaDenda>0){
				date_default_timezone_set('Asia/Jakarta');
				$jumlahDenda = $this->config->item('dendaKU') * $lamaDenda;
				$adaDenda = true;
				//tambah data yg diupdate
				$data['fine'] = $jumlahDenda;
				$data['isFine'] = $adaDenda;

				if($this->session->userdata('ownerActiveFine')){
					//insert log denda
					$denda = array(
						'loanID' => $loanID->loanID,
						'staffAccount' => $data['staffReturn'],
						'userAccount' => $loanID->userAccount,
						'libraryMainNumber' => $id,
						'loanPaid' => date('Y-m-d H:i:s'),
						'loanFee' => $jumlahDenda,
						'ownerID' => $data['ownerID']
					);
					$this->db->insert('fine_log', $denda);
				}
			}
		}//end if hari>0

		//jika ILL mengembalikan langsung di pusat oleh mhs
		if($loanID->loanType=='I' && $reloan==false){
			//kembali di pusat
			if($this->session->userdata('ownerID')==1){
				//get data order
				//orderStatus = sedang dipinjam = 7
				$whereOrder = array('userAccount' => $loanID->userAccount,
					'libraryMainNumber' => $id,
					'ownerID' => $loanID->ownerID,
					'orderStatusID' => 7
				);
				$theOrder = $this->db->get_where('ill_order', $whereOrder)->row();
				//insert trace
				//statusTrace = 2 = selesai
				$statusID = 2;
				$newTrace = array('orderID' => $theOrder->orderID,
					'statusID' => $statusID, 
					'staffID' => $this->session->userdata('userID'),
					'traceDate' => date('Y-m-d H:i:s'),
					'ownerID' => 1,
					'traceNotes' => 'KEMBALI LANGSUNG KE PUSAT'        
				);
				$this->db->insert('ill_trace', $newTrace);
				//update order ID
				$updOrderILL = array('orderStatusID' => $statusID);
				$this->db->where('orderID', $theOrder->orderID);
				$this->db->update('ill_order', $updOrderILL); 
			}
		}//end if loanType=ILL

		//proses pesan pinjam, tambah where ownerID
		$library = $this->db->get_where('library_dataunit', array('libraryMainNumber' => $loanID->libraryMainNumber, 'ownerID' => $loanID->ownerID));
		$libraryID = '';
		if($library->num_rows()){
			$libraryID = $library->row()->libraryID;
		}

		//update loan
		$this->db->where('libraryMainNumber', $id);
		$this->db->where('ownerID', $data['ownerID']);
		$this->db->where('returnedDate is null');
		$this->db->update('loan', $data);
		
		// return true;
		$dataKembali['libraryID'] = $libraryID;
		$dataKembali['sukses'] = true;
		$dataKembali['loanType'] = $loanID->loanType;

		//update frs
		$this->fungsi->updateFRS($loanID->userAccount);

		return $dataKembali;
	} else {
		// return false;
		$dataKembali['libraryID'] = '';
		$dataKembali['sukses'] = false;			
		return $dataKembali;
	}
}
```

to

```php
function retur($data, $loanID, $reloan = false)
{
	$this->union = config(\Config\Union::class);

	$q = $this->db->table('loan')
		->select('*, TO_DAYS(CURDATE())-TO_DAYS(loan.returnDate) as hari')
		->where('loanID', $loanID)
		->where('ownerID', $data['ownerID'])
		->where('returnedDate is null')
		->orderBy('loanID', 'desc')
		->limit(1);
	$loan = $q->get();

	if ($loan->getNumRows() > 0) {
		//insert denda
		$jumlahDenda = 0;
		$adaDenda = false;
		$loanID = $loan->getRow();
		if ($loanID->hari > 0) {
			$minggu = getMinggu($data['ownerID'], substr($loanID->returnDate, 0, 10), date("Y-m-d"));
			$libur = getLibur($data['ownerID'], substr($loanID->returnDate, 0, 10), date("Y-m-d"));

			$lamaDenda = $loanID->hari - $minggu - $libur;
			if ($lamaDenda > 0) {
				$jumlahDenda = $this->union->dendaKU * $lamaDenda;
				$adaDenda = true;
				//tambah data yg diupdate
				$data['fine'] = $jumlahDenda;
				$data['isFine'] = $adaDenda;
				if (session('ownerActiveFine')) {
					//insert log denda
					$denda = array(
						'loanID' => $loanID->loanID,
						'staffAccount' => $data['staffReturn'],
						'userAccount' => $loanID->userAccount,
						'libraryMainNumber' => $loanID->libraryMainNumber,
						'loanPaid' => date('Y-m-d H:i:s'),
						'loanFee' => $jumlahDenda,
						'ownerID' => $data['ownerID']
					);
					$fl = $this->db->table('fine_log');
					$fl->insert($denda);
				}
			}
		} //end if hari>0

		//jika ILL mengembalikan langsung di pusat oleh mhs
		if ($loanID->loanType == 'I' && $reloan == false) {
			//kembali di pusat
			if (session('ownerID') == 1) {
				//get data order orderStatus = sedang dipinjam = 7
				$whereOrder = array(
					'userAccount' => $loanID->userAccount,
					'libraryMainNumber' => $loanID->libraryMainNumber,
					'ownerID' => $loanID->ownerID,
					'orderStatusID' => 7
				);
				$builder = $this->db->table('ill_order');
				$theOrder = $builder->getWhere($whereOrder)->getRow();

				//insert trace statusTrace = 2 = selesai
				$statusID = 2;
				$newTrace = array(
					'orderID' => $theOrder->orderID,
					'statusID' => $statusID,
					'staffID' => $this->session->userdata('userID'),
					'traceDate' => date('Y-m-d H:i:s'),
					'ownerID' => 1,
					'traceNotes' => 'KEMBALI LANGSUNG KE PUSAT'
				);
				$illt = $this->db->table('ill_trace');
				$illt->insert($newTrace);
				//update order ID
				$updOrderILL = array('orderStatusID' => $statusID);
				$illo = $this->db->table('ill_order');
				$illo->where('orderID', $theOrder->orderID);
				$illo->update($updOrderILL);
			}
		} //end if loanType=ILL

		//proses pesan pinjam, tambah where ownerID
		$builder = $this->db->table('library_dataunit');
		$library = $builder->getWhere(['libraryMainNumber' => $loanID->libraryMainNumber, 'ownerID' => $loanID->ownerID]);
		$libraryID = '';
		if ($library->getNumRows()) {
			$libraryID = $library->getRow()->libraryID;
		}

		//update loan
		$upl = $this->db->table('loan');
		$upl->where('libraryMainNumber', $loanID->libraryMainNumber);
		$upl->where('ownerID', $data['ownerID']);
		$upl->where('returnedDate is null');
		$upl->update($data);

		// return true;
		$dataKembali['libraryID'] = $libraryID;
		$dataKembali['libraryMainNumber'] = $loanID->libraryMainNumber;
		$dataKembali['loanType'] = $loanID->loanType;
		$dataKembali['sukses'] = true;

		//update frs
		updateFRS($loanID->userAccount);

		return $dataKembali;
	} else {
		// return false;
		$dataKembali['libraryID'] = '';
		$dataKembali['libraryMainNumber'] = '';
		$dataKembali['loanType'] = '';
		$dataKembali['sukses'] = false;
		return $dataKembali;
	}
}
```

- bp (&#10003;)
- loanMail (&#10003;)
- in_reloan_staf (&#10003;)
- tutup_kasus [member]
- loanList [superuser]
- loanlist detail [superuser]
