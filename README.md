// ============================================================
// 1. KHAI BÁO BIẾN & CẤU HÌNH
// ============================================================
const connectButton = document.getElementById('connectButton');
const walletAddress = document.getElementById('walletAddress');
const btnSubmit = document.getElementById('btnSubmit');
const btnSearch = document.getElementById('btnSearch');
const recordList = document.getElementById('recordList');

// --- CẤU HÌNH CONTRACT ---
const contractAddress = "0xEC366fF1ACD45457D55e8f3851A71a77b4CA85bA"; // Địa chỉ bạn cung cấp

// ABI ĐẦY ĐỦ (Đã thêm admin và authorizeDoctor)
const abi = [
    "function addVaccination(address _patient, string memory _vaccineName, string memory _date, string memory _center) public",
    "function getRecord(address, uint256) view returns (string, string, string)",
    "function getRecordCount(address) view returns (uint256)",
    "function authorizeDoctor(address _newDoctor) public",
    "function isDoctor(address) view returns (bool)",
    "function totalSystemVaccinations() view returns (uint256)",
    "function admin() view returns (address)" // <--- QUAN TRỌNG: Để xem ai là Admin
];

// ============================================================
// 2. LOGIC DASHBOARD & ADMIN (Tự động chạy)
// ============================================================
async function loadDashboard() {
    const counterElement = document.getElementById('totalSystemCount');
    if (!counterElement) return;

    try {
        if (typeof window.ethereum !== "undefined") {
            const provider = new ethers.providers.Web3Provider(window.ethereum);
            const contract = new ethers.Contract(contractAddress, abi, provider);

            // 1. Đọc tổng số mũi tiêm
            const totalBigNum = await contract.totalSystemVaccinations();
            counterElement.innerText = totalBigNum.toString();
        }
    } catch (err) {
        console.error("Lỗi tải dashboard:", err);
        counterElement.innerText = "0";
    }
}

// --- HÀM MỚI: KIỂM TRA QUYỀN ADMIN ---
async function checkAdminRole() {
    try {
        if (typeof window.ethereum === "undefined") return;
        
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const signer = provider.getSigner();
        const contract = new ethers.Contract(contractAddress, abi, provider);

        // Lấy địa chỉ ví đang kết nối & địa chỉ Admin từ contract
        const currentAddr = await signer.getAddress();
        const adminAddr = await contract.admin();

        // So sánh (chuyển về chữ thường để tránh lỗi ký tự hoa/thường)
        if (currentAddr.toLowerCase() === adminAddr.toLowerCase()) {
            console.log("👑 PHÁT HIỆN ADMIN: " + currentAddr);
            document.getElementById('adminPanel').style.display = 'block'; // Hiện bảng đỏ
        } else {
            console.log("👤 User thường: " + currentAddr);
            document.getElementById('adminPanel').style.display = 'none'; // Ẩn bảng đỏ
        }
    } catch (err) {
        console.error("Lỗi check admin:", err);
    }
}

// --- HÀM MỚI: CẤP QUYỀN BÁC SĨ ---
async function authorizeNewDoctor() {
    const newDocAddr = document.getElementById('newDoctorAddress').value;
    if (!newDocAddr) return alert("Vui lòng nhập địa chỉ ví muốn cấp quyền!");

    try {
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const signer = provider.getSigner();
        const contract = new ethers.Contract(contractAddress, abi, signer);

        const btn = document.querySelector('#adminPanel button');
        btn.innerHTML = "Đang xác thực...";
        btn.disabled = true;

        // Gửi lệnh lên Blockchain
        const tx = await contract.authorizeDoctor(newDocAddr);
        await tx.wait(); // Chờ xác nhận

        alert("✅ Đã cấp quyền Bác sĩ thành công cho ví:\n" + newDocAddr);
        document.getElementById('newDoctorAddress').value = "";

    } catch (err) {
        console.error(err);
        alert("Lỗi cấp quyền: " + (err.reason || err.message));
    } finally {
        const btn = document.querySelector('#adminPanel button');
        btn.innerHTML = "Cấp quyền Bác sĩ";
        btn.disabled = false;
    }
}

// Gọi Dashboard ngay khi mở web
loadDashboard();

// ============================================================
// 3. ĐIỀU HƯỚNG GIAO DIỆN
// ============================================================
window.selectRole = function(role) {
    document.getElementById('roleSelection').style.display = 'none';
    document.getElementById('mainContent').style.display = 'block';
    
    if (role === 'doctor') {
        document.getElementById('roleTitle').innerText = "Khu vực Cơ sở y tế";
        document.getElementById('doctorSection').style.display = 'flex';
        document.getElementById('patientSection').style.display = 'none';
        
        // Kiểm tra lại quyền Admin khi vào giao diện bác sĩ
        checkAdminRole();
    } else {
        document.getElementById('roleTitle').innerText = "Khu vực Người dân";
        document.getElementById('doctorSection').style.display = 'none';
        document.getElementById('patientSection').style.display = 'block';
    }
}

window.backToRoles = function() {
    document.getElementById('roleSelection').style.display = 'flex';
    document.getElementById('mainContent').style.display = 'none';
    recordList.innerHTML = ""; 
}

// ============================================================
// 4. KẾT NỐI VÍ METAMASK
// ============================================================
async function connect() {
    if (typeof window.ethereum !== "undefined") {
        try {
            const accounts = await window.ethereum.request({ method: "eth_requestAccounts" });
            const shortAddr = accounts[0].substring(0, 6) + "..." + accounts[0].substring(38);
            
            walletAddress.innerHTML = `<b>Ví:</b> ${shortAddr}`;
            connectButton.className = "btn btn-success btn-sm ms-2 rounded-pill";
            connectButton.innerHTML = "Đã kết nối";
            
            // Tự điền ví vào ô tìm kiếm
            const searchInput = document.querySelector('#patientSection input[type="text"]') || document.getElementById('searchKey');
            if(searchInput) searchInput.value = accounts[0];

            // Tải lại các dữ liệu quan trọng sau khi kết nối
            loadDashboard(); 
            checkAdminRole(); // <--- Kiểm tra Admin ngay khi kết nối

        } catch (e) { alert("Kết nối bị từ chối!"); }
    } else { alert("Vui lòng cài đặt MetaMask!"); }
}

// ============================================================
// 5. CHỨC NĂNG BÁC SĨ (GHI DỮ LIỆU)
// ============================================================
async function addVaccination() {
    const patient = document.getElementById('patientAddr').value;
    const vaccine = document.getElementById('vaccineName').value;
    const today = new Date().toLocaleDateString('vi-VN'); 
    const centerName = "Bệnh viện Blockchain (Verified)";

    if (!patient || !vaccine) return alert("Vui lòng nhập đủ thông tin!");

    try {
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const signer = provider.getSigner();
        const contract = new ethers.Contract(contractAddress, abi, signer);

        btnSubmit.innerHTML = "Đang xử lý...";
        btnSubmit.disabled = true;

        const tx = await contract.addVaccination(patient, vaccine, today, centerName);
        await tx.wait(); 

        alert("✅ Ghi thành công!\nHash: " + tx.hash);
        
        document.getElementById('vaccineName').value = "";
        await loadDashboard(); 

    } catch (err) {
        console.error(err);
        if (err.message.includes("Ban khong phai bac si") || err.reason === "Ban khong phai bac si duoc cap phep!") {
            alert("❌ LỖI BẢO MẬT: Ví này không phải Bác sĩ!\n(Nếu bạn là Admin, hãy tự cấp quyền cho mình hoặc ví phụ)");
        } else {
            alert("Lỗi: " + (err.reason || err.message));
        }
    } finally {
        btnSubmit.innerHTML = "Xác nhận lên Blockchain";
        btnSubmit.disabled = false;
    }
}

// ============================================================
// 6. CHỨC NĂNG NGƯỜI DÂN: TRA CỨU
// ============================================================
async function searchRecords() {
    let searchInput = document.querySelector('#patientSection input[type="text"]') || document.getElementById('searchKey');
    
    if (!searchInput || !searchInput.value) return alert("Vui lòng nhập địa chỉ ví!");
    const patientAddress = searchInput.value.trim();

    btnSearch.innerHTML = "Đang tải...";
    btnSearch.disabled = true;
    recordList.innerHTML = ""; 

    try {
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const contract = new ethers.Contract(contractAddress, abi, provider);

        const countBigNum = await contract.getRecordCount(patientAddress);
        const count = countBigNum.toNumber();

        if (count === 0) {
            recordList.innerHTML = `<div class="alert alert-warning text-center">Ví này chưa có dữ liệu tiêm chủng!</div>`;
            return;
        }

        for (let i = 0; i < count; i++) {
            const record = await contract.getRecord(patientAddress, i);
            let linkExplorer = "https://cronos.org/explorer/testnet3/address/" + patientAddress;
            let qrId = "qr-" + i;

            let htmlCard = `
            <div class="card p-3 mb-3 shadow-sm border-start border-5 border-success">
                <div class="row align-items-center">
                    <div class="col-8">
                        <h5 class="text-success fw-bold">💉 ${record[0]}</h5>
                        <p class="mb-1 text-muted small"><strong>Ngày:</strong> ${record[1]}</p>
                        <p class="mb-1 text-muted small"><strong>Nơi cấp:</strong> ${record[2]}</p>
                        <span class="badge bg-primary">Verified on Blockchain</span>
                    </div>
                    <div class="col-4 text-center">
                        <div id="${qrId}" class="qr-container bg-white p-1 border" onclick="showLargeQR('${linkExplorer}')" style="cursor: pointer;"></div>
                        <small class="text-muted d-block mt-1" style="font-size: 10px">Click để phóng to</small>
                    </div>
                </div>
                <div class="mt-2 text-end">
                     <button class="btn btn-outline-secondary btn-sm" onclick="downloadCard(this)">⬇️ Lưu ảnh</button>
                </div>
            </div>`;
            
            recordList.insertAdjacentHTML('beforeend', htmlCard);
            new QRCode(document.getElementById(qrId), { text: linkExplorer, width: 70, height: 70 });
        }
        alert(`Tìm thấy ${count} hồ sơ!`);

    } catch (err) {
        console.error(err);
        alert("Lỗi tải dữ liệu: " + err.message);
    } finally {
        btnSearch.innerHTML = "Tìm kiếm";
        btnSearch.disabled = false;
    }
}

// ============================================================
// 7. TIỆN ÍCH (QR & DOWNLOAD)
// ============================================================
window.showLargeQR = function(url) {
    const modalContent = document.getElementById('modal-qr-content');
    if(!modalContent) return; // Tránh lỗi nếu chưa có modal
    modalContent.innerHTML = "";
    new QRCode(modalContent, { text: url, width: 300, height: 300 });
    var myModal = new bootstrap.Modal(document.getElementById('qrModal'));
    myModal.show();
}

window.downloadCard = function(btn) {
    let cardElement = btn.closest('.card');
    btn.style.display = 'none';
    
    if(typeof html2canvas !== 'undefined') {
        html2canvas(cardElement).then(canvas => {
            let link = document.createElement('a');
            link.download = 'chung-nhan-tiem.png';
            link.href = canvas.toDataURL();
            link.click();
            btn.style.display = 'inline-block';
        });
    } else {
        alert("Thiếu thư viện html2canvas! Vui lòng kiểm tra lại file index.html");
        btn.style.display = 'inline-block';
    }
}

// Gán sự kiện
if(connectButton) connectButton.onclick = connect;
if(btnSubmit) btnSubmit.onclick = addVaccination;
if(btnSearch) btnSearch.onclick = searchRecords;
