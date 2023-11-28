# Staf [&#10003;]
## Modul
- **Reset** [&#10003;]
- **Lihat Staf Non Aktif** [&#10003;]
```php
function staff_active()
{
    $active = $this->uri->segment(3);
    $this->index('', $active);
}
```
to
```php
function staff_active()
{
    $this->session->set('cachedStaffActive', true);
    return redirect()->to($this->base_ctrl);
}
```
- **Search** [&#10003;]
- **List Staf** [&#10003;]
```php
//controller
$superuser = $this->session->userdata('groupName');
$data['isSuperuser'] = FALSE;
$bCari = $this->input->post('bCari');
if ($superuser == 'superuser') {
    $data['owner'] = $this->settingmodel->table('owner', '', '', 'ownerName');
    $data['staffGroup'] = $this->settingmodel->table('staff_group', '', '', 'staffGroupName');
    $data['isSuperuser'] = TRUE;
    $where = '';
} else {
    $data['staffGroup'] = $this->settingmodel->table('staff_group', 'staffGroupName!="superuser"', '', 'staffGroupName');
    // $data['staffGroup'] = $this->settingmodel->table('staff_group', 'ownerID' == $this->session->userdata('ownerID'), '', 'staffGroupName');
    $where = ' owner.ownerID="' . $this->session->userdata('ownerID') . '" ';
}
if ($bCari != '') {
    $cari = $this->input->post('tCari');
    if ($cari != '') {
        $fields = array('staff.staffName', 'staff.staffAccount', 'owner.ownerInitial', 'owner.ownerName', 'staff_group.staffGroupName');
        $count = count($fields);
        $i = 1;
        if ($where != '') {
            $where .= ' AND ( ';
        } else {
            $where .= ' ( ';
        }

        foreach ($fields as $field) {
            if ($i == $count) {
                $where .= $field . ' like "%' . $cari . '%" ';
            } else {
                $where .= $field . ' like "%' . $cari . '%" OR ';
            }
            $i++;
        }
        $where .= ') ';
    }
}

if ($staffActive == '') {
    $staffActive = 1;
}

if ($where != '') {
    $where .= ' AND staff.staffIsActive="' . $staffActive . '" ';
} else {
    $where .= ' staff.staffIsActive="' . $staffActive . '" ';
}

$this->db->order_by('staff.staffIsActive DESC, staff.staffName ASC');
$data['staff'] = $this->staffmodel->staffList($where);
```
to
```php
$bCari = $this->request->getPost('bCari');
$superuser = $this->session->groupName;
$where = $whereConditions = '';
if ($superuser != 'superuser') {
    $where .= ' owner.ownerID = "' . $this->session->ownerID . '" ';
}

if ($bCari == 'bReset') {
    $this->session->remove('cachedStaffActive');
    return redirect()->to($this->base_ctrl);
} else {
    $cari = $this->request->getPost('tCari');
    if ($cari != '') {
        $fields = array('staff.staffName', 'staff.staffAccount', 'owner.ownerInitial', 'owner.ownerName', 'staff_group.staffGroupName');
        $whereConditions = $this->generateSearchConditions($fields, $cari);
    }
}

$cachedStaffActive = false;
if ($this->session->has('cachedStaffActive')) {
    $cachedStaffActive = $this->session->get('cachedStaffActive');
}
$staffActive = $cachedStaffActive ? 0 : 1;
$where = $this->combineConditions($where, $whereConditions, ' AND ');
$where = $this->appendStaffActiveCondition($where, $staffActive);

$page = (int) ($this->request->getGet('page') ?? 1);
$resstaf = $this->sm->staffList($where, $page, $this->perPage);
$pager_links = $this->pager->makeLinks($page, $this->perPage, $resstaf['totalRows'], 'temp_pager');
$sendData = [
    'headTable' => ['No' => 'min-w-10px', 'Staf' => 'min-w-400px', 'Akun' => 'min-w-100px', 'Perpustakaan' => 'min-w-300px', 'Aksi' => 'min-w-100px'],
    'staff' => $resstaf['results'],
    'perPage' => $this->perPage,
    'total' => $resstaf['totalRows'],
    'pager_links' => $pager_links
];
$dataDetail = ['activeView' => 'staf/widgets/list', 'data' => $sendData];
return $dataDetail;

function generateSearchConditions($fields, $cari)
{
    $where = '';
    foreach ($fields as $index => $field) {
        $condition = ($index === 0) ? '(' : ' OR ';
        $condition .= $field . ' LIKE "%' . $cari . '%" ';
        $where .= $condition;
    }
    $where .= ')';

    return $where;
}

function combineConditions($existingCondition, $newCondition, $operator)
{
    if ($newCondition) {
        $existingCondition .= ($existingCondition != '') ? $operator : '';
        $existingCondition .= $newCondition;
    }
    return $existingCondition;
}

function appendStaffActiveCondition($where, $staffActive)
{
    $where .= ($where != '') ? ' AND ' : '';
    $where .= 'staff.staffIsActive="' . $staffActive . '"';
    return $where;
}
```
- **Tambah** [&#10003;]
```php
function add()
{
    $data['login'] = $this->auth->cekLOGIN($this->modul);
    $data['title'] = $this->lang->line('add_staff');
    $data['BASE_CTRL'] = $this->base_ctrl;
    $data['activeView'] = 'staff/v_staff_add';
    $superuser = $this->session->userdata('groupName');
    $data['isSuperuser'] = FALSE;

    $data['prody'] = $this->settingmodel->table('program_study', '', '', 'programStudyName');
    if ($superuser == 'superuser') {
        $data['owner'] = $this->settingmodel->table('owner', '', '', 'ownerName');
        $data['staffGroup'] = $this->settingmodel->table('staff_group', '', '', 'staffGroupName');
        $data['isSuperuser'] = TRUE;
        $where = '';
    } else {
        $data['staffGroup'] = $this->settingmodel->table('staff_group', 'staffGroupName!="superuser"', '', 'staffGroupName');
        $where = ' owner.ownerID="' . $this->session->userdata('ownerID') . '" ';
    }
    if ($data['login']) {
        $this->load->view('v_home', $data);
    } else {
        redirect(base_url());
    }
}
```
to
```php
function addStaf()
{
    $superuser = $this->session->groupName;
    if ($superuser == 'superuser') {
        $staffGroup = $this->allModel->getT('staff_group');
    } else {
        $staffGroup = $this->allModel->getTable('staff_group', 'staffGroupName != "superuser"');
    }

    $sendData = [
        'owner' => $this->allModel->getT('owner')->getResultArray(),
        'prody' => $this->allModel->getT('program_study')->getResultArray(),
        'staffGroup' => $staffGroup->getResultArray(),
    ];
    $dataDetail = ['activeView' => 'staf/widgets/add', 'data' => $sendData];
    return $dataDetail;
}
```
- **Edit** [&#10003;]
```php
function edit()
{
    $data['login'] = $this->auth->cekLOGIN($this->modul);
    $data['title'] = $this->lang->line('edit_staff');
    $data['BASE_CTRL'] = $this->base_ctrl;
    $superuser = $this->session->userdata('groupName');

    $data['isSuperuser'] = FALSE;

    $data['prody'] = $this->settingmodel->table('program_study', '', '', 'programStudyName');
    if ($superuser == 'superuser') {
        $data['owner'] = $this->settingmodel->table('owner', '', '', 'ownerName');
        $data['staffGroup'] = $this->settingmodel->table('staff_group', '', '', 'staffGroupName');
        $data['isSuperuser'] = TRUE;
        $where = '';
    } else {
        $data['staffGroup'] = $this->settingmodel->table('staff_group', 'staffGroupName!="superuser"', '', 'staffGroupName');
        $where = ' owner.ownerID="' . $this->session->userdata('ownerID') . '" ';
    }
    if ($data['login']) {
        $data['staffID'] = $this->uri->segment(3);
        if ($data['staffID'] != '') {
            $where = array('staffID' => $data['staffID']);
            $cekStaff = $this->allmodel->getTable('staff', $where);
            if ($cekStaff->num_rows()) {
                // echo $this->db->last_query();
                $data['sg'] = $cekStaff->row_array();
                // var_dump($data['sg']);exit();
                $data['activeView'] = 'staff/v_staff_edit';
                $this->load->view('v_home', $data);
            } else {
                $this->index();
            }
        } else {
            $this->index();
        }
    } else {
        redirect(base_url());
    }
}
```
to
```php
function editStaf($staffID) //done
{
    $cekStaff = $this->sm->detailStaff($staffID);
    if ($cekStaff->getNumRows()) {
        $superuser = $this->session->groupName;
        if ($superuser == 'superuser') {
            $staffGroup = $this->allModel->getT('staff_group');
        } else {
            $staffGroup = $this->allModel->getTable('staff_group', 'staffGroupName != "superuser"');
        }

        $sendData = [
            'sDetail' => $cekStaff->getRowArray(),
            'owner' => $this->allModel->getT('owner')->getResultArray(),
            'prody' => $this->allModel->getT('program_study')->getResultArray(),
            'staffGroup' => $staffGroup->getResultArray(),
        ];
        $dataDetail = ['activeView' => 'staf/widgets/edit', 'data' => $sendData];
        return $dataDetail;
    } else {
        return null;
    }
}
```
- **Prose Add dan Edit** [&#10003;]
```php
function staff_post()
{
    $data['login'] = $this->auth->cekLOGIN($this->modul);
    $label = $this->lang->line('staff');
    $data['title'] = $this->lang->line('');

    date_default_timezone_set('Asia/Jakarta');
    $simpan = $this->input->post('bSimpan');
    // var_dump($simpan);
    // exit();
    $superuser = $this->session->userdata('groupName');
    $data['isSuperuser'] = FALSE;

    $data['prody'] = $this->settingmodel->table('program_study', '', '', 'programStudyName');
    if ($superuser == 'superuser') {
        $data['owner'] = $this->settingmodel->table('owner', '', '', 'ownerName');
        $data['staffGroup'] = $this->settingmodel->table('staff_group', '', '', 'staffGroupName');
        $data['isSuperuser'] = TRUE;
        $where = '';
    } else {
        $data['staffGroup'] = $this->settingmodel->table('staff_group', 'staffGroupName!="superuser"', '', 'staffGroupName');
        $where = ' owner.ownerID="' . $this->session->userdata('ownerID') . '" ';
    }
    // $data['activeView'] = 'staff/v_staff_add';
    // $data['BASE_CTRL'] = $this->base_ctrl;
    $this->load->library('form_validation');
    $this->form_validation->set_rules('staffName', '<b>' . $this->lang->line('staff_name') . '</b>', 'required');
    $this->form_validation->set_rules('staffAccount', '<b>' . $this->lang->line('account_INA') . '</b>', 'required');
    $this->form_validation->set_rules('staffGroupID', '<b>' . $this->lang->line('staff_group') . '</b>', 'required');
    $this->form_validation->set_rules('staffNumberID', '<b> NIP </b>', 'required');


    $staffName = trim($this->input->post('staffName'));
    $staffAccount = trim($this->input->post('staffAccount'));
    $staffNumberID = trim($this->input->post('staffNumberID'));
    $staffEmail = trim($this->input->post('staffEmail'));
    $staffEmailNonITB = trim($this->input->post('staffEmailNonITB'));
    $staffPhone = trim($this->input->post('staffPhone'));
    $staffAddress = trim($this->input->post('staffAddress'));
    $staffGroupID = $this->input->post('staffGroupID');
    $programStudyID = $this->input->post('programStudyID');
    $ownerID = $this->input->post('ownerID');
    $data['logged'] = $this->auth->cekLOGIN($this->modul);

    // $name = $this->session->userdata('ownerGroup');
    $x_filename = $_FILES['staffImage']['name'];
    $file_type = explode('.', $x_filename);
    $file_name = $staffAccount . '.' . $file_type[1];

    $path = './assets/files/photo_staff/';
    $config['upload_path']   = $path;
    $config['file_name']     = $file_name;
    $config['allowed_types'] = 'image/jpeg|jpg|jpeg|png';
    $config['max_size']     = 0;
    $config['max_width'] = 0;
    $config['max_height'] = 0;

    if ($data['logged']) {
        if ($simpan == 'Save') {

            $this->load->library('upload', $config);
            $this->upload->initialize($config);
            if (!$this->upload->do_upload('staffImage')) {
                $data['error'] = array('error' => $this->upload->display_errors());
                var_dump($data['error']);
                die;
                // if ($this->form_validation->run() == FALSE) {
                //     $pesan = array('Gagal input <b>' . $label . '</b>', 'danger');
            } else {
                //input
                $this->resizeImage($config['upload_path'] . $path);
                $uploaded_data = array('upload_data' => $this->upload->data());
                $newData = array(
                    'staffName'       => $staffName,
                    'staffAccount'    => $staffAccount,
                    'staffNumberID'   => $staffNumberID,
                    'staffEmail'      => $staffEmail,
                    'staffEmailNonITB' => $staffEmailNonITB,
                    'staffImagePath' => $path . $uploaded_data['upload_data']["file_name"],
                    'staffPhone'      => $staffPhone,
                    'staffAddress'    => $staffAddress,
                    'staffGroupID'    => $staffGroupID,
                    'programStudyID'  => $programStudyID,
                    'ownerID'         => $ownerID,
                    'staffInsertDate'   => date('Y-m-d H:i:s'),
                    'staffInsertID' => $this->session->userdata('userID')
                );
                $pesan = $this->staffmodel->insert($newData);
            }
            $this->index($pesan);
        } else if ($simpan == 'UpdateStaff') {
            $id = $this->input->post('staffID');
            $gbFilesize = $_FILES['staffImage']['size'];
            if ($gbFilesize > 0) {
                $x_filename = $_FILES['staffImage']['name'];
                $config['file_name'] = $file_name;
                $config['overwrite'] = true;
                $this->load->library('upload', $config);
                if (!$this->upload->do_upload('staffImage')) {
                    $error = array('error' => $this->upload->display_errors());
                    echo var_dump($error);
                    die;
                } else {
                    $this->resizeImage($config['upload_path'] . $path);
                    $uploaded_data = array('upload_data' => $this->upload->data());
                $updData = array(
                    'staffName'     => $staffName,
                    'staffAccount'  => $staffAccount,
                    'staffNumberID' => $staffNumberID,
                    'staffEmail'    => $staffEmail,
                    'staffEmailNonITB' => $staffEmailNonITB,
                    'staffImagePath' => $path . $uploaded_data['upload_data']["file_name"],
                    'staffPhone'    => $staffPhone,
                    'staffAddress'  => $staffAddress,
                    'staffGroupID'  => $staffGroupID,
                    'programStudyID'  => $programStudyID,
                    'ownerID'       => $ownerID,
                    'staffEditDate'   => date('Y-m-d H:i:s'),
                    'staffEditID' => $this->session->userdata('userID')
                );
                $this->staffmodel->update($id, $updData);
                // var_dump($updData);
                // exit();
                $pesan = array('Berhasil update ' . $this->lang->line('staff') . ' <b>' . $staffName . '</b>', 'info');
                $this->index($pesan);
                }

            }else{
                $updData = array(
                    'staffName'     => $staffName,
                    'staffAccount'  => $staffAccount,
                    'staffNumberID' => $staffNumberID,
                    'staffEmail'    => $staffEmail,
                    'staffEmailNonITB' => $staffEmailNonITB,
                    // 'staffImagePath' => $path . $uploaded_data['upload_data']["file_name"],
                    'staffPhone'    => $staffPhone,
                    'staffAddress'  => $staffAddress,
                    'staffGroupID'  => $staffGroupID,
                    'programStudyID'  => $programStudyID,
                    'ownerID'       => $ownerID,
                    'staffEditDate'   => date('Y-m-d H:i:s'),
                    'staffEditID' => $this->session->userdata('userID')
                );
                $this->staffmodel->update($id, $updData);
                // var_dump($updData);
                // exit();
                $pesan = array('Berhasil update ' . $this->lang->line('staff') . ' <b>' . $staffName . '</b>', 'info');
                $this->index($pesan);
            }
                //update
                
            // }
        }
    }
    //end if login
    else {
        $this->index();
    }
}
```
to
```php
function proses()
{
    $validation = \Config\Services::validation();
    $req = $this->request->getPost();
    $btn = trim($req['bSimpan']);
    $itbAccount = trim($req['itbAccount']);
    $staffName = trim($req['staffName']);
    $staffAccount = trim($req['staffAccount']);
    $staffNumberID = trim($req['staffNumberID']);
    $staffEmail = trim($req['staffEmail']);
    $staffEmailNonITB = trim($req['staffEmailNonITB']);
    $staffPhone = trim($req['staffPhone']);
    $staffAddress = trim($req['staffAddress']);
    $staffGroupID = $req['staffGroupID'];
    $programStudyID = $req['programStudyID'];
    $ownerID = $req['ownerID'];
    $file = $this->request->getFile('staffImage');

    $validation->setRules([
        'staffImage' => 'max_size[staffImage,2048]|ext_in[staffImage,jpeg,jpg,png]'
    ]);

    if ($validation->run() == FALSE) {
        return redirect()->back()->withInput()->with('pesan', ['danger', $validation->getError('staffImage')]);
    } else {
        $upload_location = "./assets/files/photo_staff/";
        if ($btn == 'Save') {
            if ($file->isValid() && !$file->hasMoved()) {
                $ext = $file->getClientExtension();
                $file_name = $staffAccount . '.' . $ext;
                $file->move($upload_location, $file_name);
                $this->resizeImage($upload_location, $file_name);
            } else {
                $file_name = 'blank.png';
            }

            $newData = array(
                'itbAccount'      => $itbAccount,
                'staffName'       => $staffName,
                'staffAccount'    => $staffAccount,
                'staffNumberID'   => $staffNumberID,
                'staffEmail'      => $staffEmail,
                'staffEmailNonITB' => $staffEmailNonITB,
                'staffImagePath' => $upload_location . $file_name,
                'staffPhone'      => $staffPhone,
                'staffAddress'    => $staffAddress,
                'staffGroupID'    => $staffGroupID,
                'programStudyID'  => $programStudyID,
                'ownerID'         => $ownerID,
                'staffInsertDate'   => date('Y-m-d H:i:s'),
                'staffInsertID' => $this->session->userID
            );
            $lastInID = $this->allModel->addT('staff', $newData);
            return redirect()->to($this->base_ctrl . '?q=read&id=' . $lastInID)->with('pesan', ['success', 'Berhasil Input <b>' . $staffName . '</b> Sebagai Staf Perpustakaan']);
        } else if ($btn == 'UpdateStaff') {
            $staffID = trim($req['staffID']);
            if ($file->isValid() && !$file->hasMoved()) {
                $ext = $file->getClientExtension();
                $file_name = $staffAccount . '.' . $ext;
                $file->move($upload_location, $file_name, true);
                $this->resizeImage($upload_location, $file_name);
                $staffImagePathAsli = $upload_location . $file_name;
            } else {
                $old_img_path = trim($req['staffImagePath']);
                $avatar_remove = trim($req['avatar_remove']);
                if (file_exists($old_img_path) && $avatar_remove == '') {
                    $staffImagePathAsli = $old_img_path;
                } else {
                    $file_name = 'blank.png';
                    $staffImagePathAsli = $upload_location . $file_name;
                }
            }

            $newDataUp = array(
                'itbAccount'      => $itbAccount,
                'staffName'       => $staffName,
                'staffAccount'    => $staffAccount,
                'staffNumberID'   => $staffNumberID,
                'staffEmail'      => $staffEmail,
                'staffEmailNonITB' => $staffEmailNonITB,
                'staffImagePath' => $staffImagePathAsli,
                'staffPhone'      => $staffPhone,
                'staffAddress'    => $staffAddress,
                'staffGroupID'    => $staffGroupID,
                'programStudyID'  => $programStudyID,
                'ownerID'         => $ownerID,
                'staffEditDate'   => date('Y-m-d H:i:s'),
                'staffEditID' => $this->session->userID
            );
            $whr = ['staffID' => $staffID];
            $this->allModel->edit('staff', $whr, $newDataUp);

            //set session baru jika user yg sama update data
            if ($this->session->userID == $staffID) {
                $strModulAkses = $this->allModel->getTable('staff_privileges', array('staffGroupID' => $staffGroupID))->getRow()->fileAccessed;
                $groupName = $this->allModel->getTable('staff_group', array('staffGroupID' => $staffGroupID))->getRow()->staffGroupName;
                $cekOwner = $this->allModel->getTable('owner', array('ownerID' => $ownerID))->getRowArray();
                $old_ses = ['itbAccount', 'userName', 'userPhoto', 'userEmail', 'userITBNumber', 'ownerID', 'ownerInitial', 'ownerAllowNIM', 'ownerActiveFine', 'ownerHoliday', 'orientationPolicy', 'groupName', 'userModulAccess', 'userGroupID', 'prodyID'];
                $this->session->remove($old_ses);
                $new_ses = [
                    'itbAccount' => $itbAccount,
                    'userName' => $staffName,
                    'userPhoto' => $staffImagePathAsli,
                    'userEmail' => $staffEmail,
                    'userITBNumber' => $staffNumberID,
                    'ownerID' => $ownerID,
                    'ownerInitial' => $cekOwner['ownerInitial'],
                    'ownerAllowNIM' => $cekOwner['ownerAllowNIM'],
                    'ownerActiveFine' => $cekOwner['ownerActiveFine'],
                    'ownerHoliday' => $cekOwner['ownerHoliday'],
                    'orientationPolicy' => $cekOwner['ownerOrientationPolicy'],
                    'groupName' => $groupName,
                    'userModulAccess' => $strModulAkses,
                    'userGroupID' => $staffGroupID,
                    'prodyID' => $programStudyID
                ];
                $this->session->set($new_ses);
            }

            return redirect()->to($this->base_ctrl . '?q=read&id=' . $staffID)->with('pesan', ['success', 'Berhasil Update <b>' . $staffName . '</b>']);
        }
    }
}
```
- **Detail** [&#10003;]
- **Non Aktifkan** [&#10003;]
- **Tambahan Set Session baru jika edit diriinya sendiri** [&#10003;]
```php
if ($this->session->userID == $staffID) {
    $strModulAkses = $this->allModel->getTable('staff_privileges', array('staffGroupID' => $staffGroupID))->getRow()->fileAccessed;
    $groupName = $this->allModel->getTable('staff_group', array('staffGroupID' => $staffGroupID))->getRow()->staffGroupName;
    $cekOwner = $this->allModel->getTable('owner', array('ownerID' => $ownerID))->getRowArray();
    $old_ses = ['itbAccount', 'userName', 'userPhoto', 'userEmail', 'userITBNumber', 'ownerID', 'ownerInitial', 'ownerAllowNIM', 'ownerActiveFine', 'ownerHoliday', 'orientationPolicy', 'groupName', 'userModulAccess', 'userGroupID', 'prodyID'];
    $this->session->remove($old_ses);
    $new_ses = [
        'itbAccount' => $itbAccount,
        'userName' => $staffName,
        'userPhoto' => $staffImagePathAsli,
        'userEmail' => $staffEmail,
        'userITBNumber' => $staffNumberID,
        'ownerID' => $ownerID,
        'ownerInitial' => $cekOwner['ownerInitial'],
        'ownerAllowNIM' => $cekOwner['ownerAllowNIM'],
        'ownerActiveFine' => $cekOwner['ownerActiveFine'],
        'ownerHoliday' => $cekOwner['ownerHoliday'],
        'orientationPolicy' => $cekOwner['ownerOrientationPolicy'],
        'groupName' => $groupName,
        'userModulAccess' => $strModulAkses,
        'userGroupID' => $staffGroupID,
        'prodyID' => $programStudyID
    ];
    $this->session->set($new_ses);
}

return redirect()->to($this->base_ctrl . '?q=read&id=' . $staffID)->with('pesan', ['success', 'Berhasil Update <b>' . $staffName . '</b>']);
```
- **Resize Image** [&#10003;]
```php
//tapi jadi jelek kualitasnya
function resizeImage($filePathName, $name) //done
{
    $image = \Config\Services::image();
    try {
        $image->withFile($filePathName . $name)
            ->resize(450, 500, true, 'height')
            ->save($filePathName . $name, 100);
    } catch (CodeIgniter\Images\Exceptions\ImageException $e) {
        var_dump($e->getMessage());
        die;
    }
}
```
