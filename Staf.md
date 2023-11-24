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

//tidak perlu set uri statActive 1, ci4 aga ribet routing match get and post 1 control
$cachedStaffActive = false;
if ($this->session->has('cachedStaffActive')) {
    $cachedStaffActive = $this->session->get('cachedStaffActive');
}
$staffActive = $cachedStaffActive ? 0 : 1;
$where = $this->combineConditions($where, $whereConditions, ' AND ');

$where = $this->appendStaffActiveCondition($where, $staffActive);

$page = (int) ($this->request->getGet('page') ?? 1);
$resstaf = $this->sm->staffList($where, $page, $this->perPage);
```
- **Tambah Staf**
- **Aksi**
