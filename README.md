#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <math.h>

/* ====================== KHAI BAO DU LIEU ======================
 * Các hằng số xác định tên file, kích thước chuỗi và những tham số
 * dùng trong công thức tính lương minh họa.
 */
#define FILE_NAME "nhanvien.dat"
#define CODE_SIZE 20
#define NAME_SIZE 100
#define DEPT_SIZE 50
#define ROLE_SIZE 50

/* Các con số này chỉ để minh họa, không phải khẳng định mức thuế/phí hiện hành. */
#define INSURANCE_RATE 0.08
#define PERSONAL_DEDUCTION 11000000.0
#define DEMO_TAX_RATE 0.05
#define STANDARD_WORK_DAYS 26.0
#define WORK_HOURS_PER_DAY 8.0
#define OVERTIME_MULTIPLIER 1.5

typedef struct {
    char maNV[CODE_SIZE];
    char hoTen[NAME_SIZE];
    char phongBan[DEPT_SIZE];
    char chucVu[ROLE_SIZE];

    double luongCoBan;
    double phuCap;

    /* Số liệu chấm công cho tháng đang được quản lý */
    double ngayCong; // ngày công
    double gioTangCa;// giờ tăng ca
} NhanVien; // thông tin của nhân viên

typedef struct Node {
    NhanVien data;
    struct Node *next;
} Node;

typedef struct {
    char magic[8];
    int version;
    int thang;
    int nam;
    unsigned int soNhanVien;
} FileHeader;

/* ====================== NHAP VA KIEM TRA DU LIEU ======================
 * Các hàm đọc dữ liệu từ bàn phím đều trả về 0 nếu không thể tiếp tục
 * nhập (ví dụ gặp EOF); giá trị nhập chỉ được chấp nhận khi đúng định dạng.
 */
/* Đọc một dòng, không để lại ký tự thừa trong bộ đệm nhập. */
static int docDong(const char *prompt, char *buffer, size_t capacity) {
    size_t length;
    int ch;

    for (;;) { // vòng lặp vô hạn
        printf("%s", prompt);

        if (fgets(buffer, (int)capacity, stdin) == NULL) {
            return 0;
        }

        length = strlen(buffer);

        if (length > 0 && buffer[length - 1] == '\n') {
            buffer[length - 1] = '\0';
            return 1;
        }

        if (feof(stdin)) {
            return 1;
        }

        /* Dòng quá dài: bỏ phần còn lại rồi yêu cầu nhập lại. */
        while ((ch = getchar()) != '\n' && ch != EOF) {
        }

        printf("Du lieu qua dai, vui long nhap lai.\n");
    }
}

static int docChuoi(const char *prompt, char *buffer, size_t capacity) {
    for (;;) {
        if (!docDong(prompt, buffer, capacity)) {
            return 0;
        }

        if (buffer[0] != '\0') {
            return 1;
        }

        printf("Khong duoc de trong.\n");
    }
}

static int docSoNguyen(const char *prompt, int minValue, int maxValue, int *result) {
    char buffer[100];
    char *end;
    long value;

    for (;;) {
        if (!docDong(prompt, buffer, sizeof(buffer))) {
            return 0;
        }

        errno = 0;
        value = strtol(buffer, &end, 10);

        while (*end == ' ' || *end == '\t') {
            end++;
        }

        if (errno == 0 && end != buffer && *end == '\0'
            && value >= minValue && value <= maxValue) {
            *result = (int)value;
            return 1;
        }

        printf("Vui long nhap so nguyen tu %d den %d.\n", minValue, maxValue);
    }
}

static int docSoThuc(const char *prompt, double minValue, double maxValue,
                     double *result) {
    char buffer[100];
    char *end;
    double value;

    for (;;) {
        if (!docDong(prompt, buffer, sizeof(buffer))) {
            return 0;
        }

        errno = 0;
        value = strtod(buffer, &end);

        while (*end == ' ' || *end == '\t') {
            end++;
        }

        if (errno == 0 && end != buffer && *end == '\0'
            && isfinite(value) && value >= minValue && value <= maxValue) {
            *result = value;
            return 1;
        }

        printf("Gia tri khong hop le. Vui long nhap tu %.2f den %.2f.\n",
               minValue, maxValue);
    }
}

static int docXacNhan(const char *prompt) {
    char buffer[20];

    for (;;) {
        if (!docDong(prompt, buffer, sizeof(buffer))) {
            return 0;
        }

        if (strcmp(buffer, "y") == 0 || strcmp(buffer, "Y") == 0) {
            return 1;
        }

        if (strcmp(buffer, "n") == 0 || strcmp(buffer, "N") == 0) {
            return 0;
        }

        printf("Nhap y de dong y hoac n de huy.\n");
    }
}

/* ====================== QUAN LY DANH SACH LIEN KET ======================
 * Tìm nhân viên theo mã, đếm số node và giải phóng toàn bộ bộ nhớ
 * trước khi chương trình kết thúc hoặc khi dữ liệu đọc bị lỗi.
 */
static Node *timNodeTheoMa(Node *head, const char *maNV) {
    while (head != NULL) {
        if (strcmp(head->data.maNV, maNV) == 0) {
            return head;
        }
        head = head->next;
    }

    return NULL;
}

static int demNhanVien(const Node *head) {
    int count = 0;

    while (head != NULL) {
        count++;
        head = head->next;
    }

    return count;
}

static void giaiPhongDanhSach(Node *head) {
    while (head != NULL) {
        Node *next = head->next;
        free(head);
        head = next;
    }
}

/* ====================== TINH LUONG VA HIEN THI HO SO ======================
 * Tách các phép tính thành hàm riêng để dùng nhất quán khi lập bảng lương,
 * sắp xếp và lọc nhân viên. Công thức là số liệu minh họa ở phần hằng số.
 */
/*
 * Công thức minh họa:
 *   Lương theo ngày công = lương cơ bản * ngày công / 26
 *   Tiền tăng ca = lương cơ bản / 26 / 8 * 1.5 * số giờ tăng ca
 *   Lương gộp = lương theo ngày công + phụ cấp + tiền tăng ca
 *   Bảo hiểm = lương cơ bản * 8%
 *   Thuế minh họa = 5% * phần thu nhập chịu thuế sau giảm trừ
 */
static double tinhLuongGop(const NhanVien *nv) {
    double luongTheoNgayCong;
    double tienTangCa;

    luongTheoNgayCong =
        nv->luongCoBan * nv->ngayCong / STANDARD_WORK_DAYS;

    tienTangCa =
        nv->luongCoBan / STANDARD_WORK_DAYS / WORK_HOURS_PER_DAY
        * OVERTIME_MULTIPLIER * nv->gioTangCa;

    return luongTheoNgayCong + nv->phuCap + tienTangCa;
}

static double tinhBaoHiem(const NhanVien *nv) {
    return nv->luongCoBan * INSURANCE_RATE;
}

static double tinhThue(const NhanVien *nv) {
    double thuNhapChiuThue =
        tinhLuongGop(nv) - tinhBaoHiem(nv) - PERSONAL_DEDUCTION;

    if (thuNhapChiuThue < 0.0) {
        thuNhapChiuThue = 0.0;
    }

    return thuNhapChiuThue * DEMO_TAX_RATE;
}

static double tinhThucLinh(const NhanVien *nv) {
    return tinhLuongGop(nv) - tinhBaoHiem(nv) - tinhThue(nv);
}

static void inThongTinNhanVien(const NhanVien *nv) {
    printf("Ma NV       : %s\n", nv->maNV);
    printf("Ho ten      : %s\n", nv->hoTen);
    printf("Phong ban   : %s\n", nv->phongBan);
    printf("Chuc vu     : %s\n", nv->chucVu);
    printf("Luong co ban: %.0f\n", nv->luongCoBan);
    printf("Phu cap     : %.0f\n", nv->phuCap);
    printf("Ngay cong   : %.1f\n", nv->ngayCong);
    printf("Gio tang ca : %.1f\n", nv->gioTangCa);
}

static void nhapHoSo(NhanVien *nv) {
    docChuoi("Ma nhan vien: ", nv->maNV, sizeof(nv->maNV));
    docChuoi("Ho ten: ", nv->hoTen, sizeof(nv->hoTen));
    docChuoi("Phong ban: ", nv->phongBan, sizeof(nv->phongBan));
    docChuoi("Chuc vu: ", nv->chucVu, sizeof(nv->chucVu));
    docSoThuc("Luong co ban: ", 0.0, 1000000000000.0, &nv->luongCoBan);
    docSoThuc("Phu cap: ", 0.0, 1000000000000.0, &nv->phuCap);
}

/* ====================== THEM, SUA, XOA NHAN VIEN ======================
 * Các thao tác cập nhật danh sách liên kết. Mã nhân viên được dùng làm
 * khóa tìm kiếm và phải duy nhất; sửa hồ sơ giữ nguyên dữ liệu chấm công.
 */
static void themNhanVien(Node **head) {
    NhanVien nv;
    Node *nodeMoi;
    Node *cuoi;

    printf("\n--- Them nhan vien ---\n");
    nhapHoSo(&nv);

    if (timNodeTheoMa(*head, nv.maNV) != NULL) {
        printf("Ma nhan vien da ton tai.\n");
        return;
    }

    nv.ngayCong = 0.0;
    nv.gioTangCa = 0.0;

    nodeMoi = malloc(sizeof(*nodeMoi));
    if (nodeMoi == NULL) {
        fprintf(stderr, "Khong du bo nho de them nhan vien.\n");
        return;
    }

    nodeMoi->data = nv;
    nodeMoi->next = NULL;

    if (*head == NULL) {
        *head = nodeMoi;
    } else {
        cuoi = *head;
        while (cuoi->next != NULL) {
            cuoi = cuoi->next;
        }
        cuoi->next = nodeMoi;
    }

    printf("Da them nhan vien.\n");
}

static void suaNhanVien(Node *head) {
    char maNV[CODE_SIZE];
    Node *node;
    NhanVien thongTinMoi;
    double ngayCongCu;
    double gioTangCaCu;

    printf("\n--- Sua nhan vien ---\n");
    if (!docChuoi("Nhap ma nhan vien can sua: ", maNV, sizeof(maNV))) {
        return;
    }

    node = timNodeTheoMa(head, maNV);
    if (node == NULL) {
        printf("Khong tim thay nhan vien.\n");
        return;
    }

    ngayCongCu = node->data.ngayCong;
    gioTangCaCu = node->data.gioTangCa;

    nhapHoSo(&thongTinMoi);

    if (strcmp(thongTinMoi.maNV, maNV) != 0
        && timNodeTheoMa(head, thongTinMoi.maNV) != NULL) {
        printf("Ma nhan vien moi da duoc su dung. Khong cap nhat.\n");
        return;
    }

    thongTinMoi.ngayCong = ngayCongCu;
    thongTinMoi.gioTangCa = gioTangCaCu;
    node->data = thongTinMoi;

    printf("Da cap nhat nhan vien.\n");
}

static void xoaNhanVien(Node **head) {
    char maNV[CODE_SIZE];
    Node *current = *head;
    Node *previous = NULL;

    printf("\n--- Xoa nhan vien ---\n");
    if (!docChuoi("Nhap ma nhan vien can xoa: ", maNV, sizeof(maNV))) {
        return;
    }

    while (current != NULL && strcmp(current->data.maNV, maNV) != 0) {
        previous = current;
        current = current->next;
    }

    if (current == NULL) {
        printf("Khong tim thay nhan vien.\n");
        return;
    }

    if (!docXacNhan("Xac nhan xoa? (y/n): ")) {
        printf("Da huy thao tac.\n");
        return;
    }

    if (previous == NULL) {
        *head = current->next;
    } else {
        previous->next = current->next;
    }

    free(current);
    printf("Da xoa nhan vien.\n");
}

/* ====================== TRA CUU VA CHAM CONG ======================
 * Hiển thị danh sách/hồ sơ theo yêu cầu, hoặc cập nhật ngày công và
 * giờ tăng ca cho nhân viên trong tháng hiện tại.
 */
static void inDanhSach(const Node *head) {
    if (head == NULL) {
        printf("Danh sach nhan vien dang rong.\n");
        return;
    }

    printf("\n%-12s %-24s %-18s %-16s %14s %14s\n",
           "Ma NV", "Ho ten", "Phong ban", "Chuc vu",
           "Luong co ban", "Phu cap");
    printf("-----------------------------------------------------------------------------------------------\n");

    while (head != NULL) {
        printf("%-12s %-24.24s %-18.18s %-16.16s %14.0f %14.0f\n",
               head->data.maNV,
               head->data.hoTen,
               head->data.phongBan,
               head->data.chucVu,
               head->data.luongCoBan,
               head->data.phuCap);
        head = head->next;
    }
}

static void timVaInNhanVien(Node *head) {
    char maNV[CODE_SIZE];
    Node *node;

    printf("\n--- Tim nhan vien ---\n");
    if (!docChuoi("Nhap ma nhan vien: ", maNV, sizeof(maNV))) {
        return;
    }

    node = timNodeTheoMa(head, maNV);
    if (node == NULL) {
        printf("Khong tim thay nhan vien.\n");
        return;
    }

    inThongTinNhanVien(&node->data);
}

static void nhapChamCong(Node *head) {
    char maNV[CODE_SIZE];
    Node *node;

    printf("\n--- Nhap cham cong ---\n");
    if (!docChuoi("Nhap ma nhan vien: ", maNV, sizeof(maNV))) {
        return;
    }

    node = timNodeTheoMa(head, maNV);
    if (node == NULL) {
        printf("Khong tim thay nhan vien.\n");
        return;
    }

    docSoThuc("So ngay cong (0-31): ", 0.0, 31.0, &node->data.ngayCong);
    docSoThuc("So gio tang ca (0-300): ", 0.0, 300.0, &node->data.gioTangCa);

    printf("Da cap nhat cham cong.\n");
}

/* ====================== BAO CAO, SAP XEP VA LOC ======================
 * Lập bảng lương và tổng thực lĩnh, sắp xếp giảm dần theo thực lĩnh,
 * hoặc chỉ hiển thị nhân viên đạt ngưỡng lương do người dùng nhập.
 */
static void inBangLuong(const Node *head, int thang, int nam) {
    double tongThucLinh = 0.0;

    if (head == NULL) {
        printf("Danh sach nhan vien dang rong.\n");
        return;
    }

    printf("\nBANG LUONG THANG %02d/%d\n", thang, nam);
    printf("%-12s %-22s %10s %10s %12s %12s %12s %14s\n",
           "Ma NV", "Ho ten", "Ngay cong", "Tang ca",
           "Luong gop", "Bao hiem", "Thue", "Thuc linh");
    printf("---------------------------------------------------------------------------------------------------------------\n");

    while (head != NULL) {
        const NhanVien *nv = &head->data;
        double luongGop = tinhLuongGop(nv);
        double baoHiem = tinhBaoHiem(nv);
        double thue = tinhThue(nv);
        double thucLinh = tinhThucLinh(nv);

        printf("%-12s %-22.22s %10.1f %10.1f %12.0f %12.0f %12.0f %14.0f\n",
               nv->maNV, nv->hoTen, nv->ngayCong, nv->gioTangCa,
               luongGop, baoHiem, thue, thucLinh);

        tongThucLinh += thucLinh;
        head = head->next;
    }

    printf("---------------------------------------------------------------------------------------------------------------\n");
    printf("Tong thuc linh: %.0f\n", tongThucLinh);
}

static void sapXepTheoLuong(Node *head) {
    int daDoi;
    Node *current;

    if (head == NULL || head->next == NULL) {
        printf("Can it nhat 2 nhan vien de sap xep.\n");
        return;
    }

    /* Sắp xếp giảm dần theo thực lĩnh bằng cách đổi dữ liệu giữa các node. */
    do {
        daDoi = 0;
        current = head;

        while (current->next != NULL) {
            if (tinhThucLinh(&current->data)
                < tinhThucLinh(&current->next->data)) {
                NhanVien tam = current->data;
                current->data = current->next->data;
                current->next->data = tam;
                daDoi = 1;
            }
            current = current->next;
        }
    } while (daDoi);

    printf("Da sap xep giam dan theo thuc linh.\n");
}

static void locTheoThucLinh(const Node *head) {
    double mucLuong;
    int timThay = 0;

    if (!docSoThuc("Nhap muc thuc linh toi thieu: ",
                   0.0, 1000000000000.0, &mucLuong)) {
        return;
    }

    printf("\nNhan vien co thuc linh tu %.0f tro len:\n", mucLuong);

    while (head != NULL) {
        if (tinhThucLinh(&head->data) >= mucLuong) {
            printf("%-12s %-24s Thuc linh: %.0f\n",
                   head->data.maNV,
                   head->data.hoTen,
                   tinhThucLinh(&head->data));
            timThay = 1;
        }
        head = head->next;
    }

    if (!timThay) {
        printf("Khong co nhan vien nao dat muc nay.\n");
    }
}

/* ====================== LUU VA DOC DU LIEU ======================
 * File nhị phân gồm phần đầu FileHeader (định dạng, tháng/năm, số lượng)
 * theo sau là từng NhanVien; không lưu con trỏ next của danh sách liên kết.
 */
static int luuDuLieu(const char *tenFile, const Node *head, int thang, int nam) {
    FILE *file = fopen(tenFile, "wb");
    FileHeader header;
    const Node *current;

    if (file == NULL) {
        perror("Khong mo duoc file de ghi");
        return 0;
    }

    memset(&header, 0, sizeof(header));
    memcpy(header.magic, "DT07DATA", 8);
    header.version = 1;
    header.thang = thang;
    header.nam = nam;
    header.soNhanVien = (unsigned int)demNhanVien(head);

    if (fwrite(&header, sizeof(header), 1, file) != 1) {
        perror("Loi khi ghi phan dau file");
        fclose(file);
        return 0;
    }

    current = head;
    while (current != NULL) {
        /* Chỉ ghi struct NhanVien, không ghi node có con trỏ next. */
        if (fwrite(&current->data, sizeof(NhanVien), 1, file) != 1) {
            perror("Loi khi ghi nhan vien");
            fclose(file);
            return 0;
        }
        current = current->next;
    }

    if (fclose(file) != 0) {
        perror("Loi khi dong file");
        return 0;
    }

    printf("Da luu %u nhan vien vao file %s.\n", header.soNhanVien, tenFile);
    return 1;
}

/* Đọc vào danh sách tạm; chỉ thay danh sách hiện tại sau khi đọc đủ dữ liệu
 * hợp lệ, để tránh làm mất dữ liệu đang có nếu file bị lỗi giữa chừng.
 */
static int docDuLieu(const char *tenFile, Node **head, int *thang, int *nam) {
    FILE *file = fopen(tenFile, "rb");
    FileHeader header;
    Node *danhSachMoi = NULL;
    Node *nodeCuoi = NULL;
    unsigned int i;

    if (file == NULL) {
        if (errno != ENOENT) {
            perror("Khong mo duoc file de doc");
        }
        return 0;
    }

    if (fread(&header, sizeof(header), 1, file) != 1
        || memcmp(header.magic, "DT07DATA", 8) != 0
        || header.version != 1
        || header.thang < 1 || header.thang > 12
        || header.nam < 2000 || header.nam > 9999
        || header.soNhanVien > 100000) {
        fprintf(stderr, "File du lieu khong hop le hoac khong dung dinh dang DT07.\n");
        fclose(file);
        return -1;
    }

    for (i = 0; i < header.soNhanVien; i++) {
        NhanVien nv;
        Node *nodeMoi;

        if (fread(&nv, sizeof(nv), 1, file) != 1) {
            fprintf(stderr, "File bi thieu du lieu nhan vien.\n");
            fclose(file);
            giaiPhongDanhSach(danhSachMoi);
            return -1;
        }

        if (memchr(nv.maNV, '\0', sizeof(nv.maNV)) == NULL
            || memchr(nv.hoTen, '\0', sizeof(nv.hoTen)) == NULL
            || memchr(nv.phongBan, '\0', sizeof(nv.phongBan)) == NULL
            || memchr(nv.chucVu, '\0', sizeof(nv.chucVu)) == NULL) {
            fprintf(stderr, "Du lieu chuoi trong file khong hop le.\n");
            fclose(file);
            giaiPhongDanhSach(danhSachMoi);
            return -1;
        }

        nodeMoi = malloc(sizeof(*nodeMoi));
        if (nodeMoi == NULL) {
            fprintf(stderr, "Khong du bo nho de doc danh sach.\n");
            fclose(file);
            giaiPhongDanhSach(danhSachMoi);
            return -1;
        }

        nodeMoi->data = nv;
        nodeMoi->next = NULL;

        if (danhSachMoi == NULL) {
            danhSachMoi = nodeMoi;
        } else {
            nodeCuoi->next = nodeMoi;
        }
        nodeCuoi = nodeMoi;
    }

    if (ferror(file)) {
        fprintf(stderr, "Co loi khi doc file.\n");
        fclose(file);
        giaiPhongDanhSach(danhSachMoi);
        return -1;
    }

    fclose(file);

    *head = danhSachMoi;
    *thang = header.thang;
    *nam = header.nam;
    return 1;
}

/* ====================== QUAN LY THANG CHAM CONG ======================
 * Chuyển tháng/năm sau khi xác nhận và xóa ngày công, giờ tăng ca cũ
 * để dữ liệu chấm công bắt đầu lại cho kỳ mới.
 */
static void doiThang(Node *head, int *thang, int *nam) {
    int thangMoi;
    int namMoi;

    printf("\nDoi thang se xoa so ngay cong va gio tang ca hien tai.\n");
    if (!docXacNhan("Ban co muon tiep tuc? (y/n): ")) {
        printf("Da huy thao tac.\n");
        return;
    }

    if (!docSoNguyen("Nhap thang moi (1-12): ", 1, 12, &thangMoi)
        || !docSoNguyen("Nhap nam moi (2000-9999): ", 2000, 9999, &namMoi)) {
        return;
    }

    while (head != NULL) {
        head->data.ngayCong = 0.0;
        head->data.gioTangCa = 0.0;
        head = head->next;
    }

    *thang = thangMoi;
    *nam = namMoi;
    printf("Da doi sang thang %02d/%d va xoa cham cong cu.\n", *thang, *nam);
}

/* In các lựa chọn thao tác cùng tháng/năm đang được quản lý. */
static void hienThiMenu(int thang, int nam) {
    printf("\n========================================\n");
    printf(" QUAN LY NHAN SU - THANG %02d/%d\n", thang, nam);
    printf("========================================\n");
    printf("1. Them nhan vien\n");
    printf("2. Sua nhan vien\n");
    printf("3. Xoa nhan vien\n");
    printf("4. Hien thi danh sach nhan vien\n");
    printf("5. Tim nhan vien theo ma\n");
    printf("6. Nhap/cap nhat cham cong\n");
    printf("7. Tinh va hien thi bang luong\n");
    printf("8. Sap xep theo thuc linh giam dan\n");
    printf("9. Loc nhan vien theo muc thuc linh\n");
    printf("10. Doi thang cham cong\n");
    printf("11. Luu du lieu\n");
    printf("0. Luu va thoat\n");
}

/* ====================== HAM MAIN - DIEU PHOI CHUONG TRINH ======================
 * Nạp dữ liệu đã lưu (hoặc khởi tạo kỳ chấm công mới), lặp menu để gọi
 * chức năng tương ứng, rồi lưu dữ liệu và giải phóng danh sách khi thoát.
 */
int main(void) {
    Node *danhSach = NULL;
    int thang = 1;
    int nam = 2026;
    int ketQuaDoc;
    int luaChon;

    ketQuaDoc = docDuLieu(FILE_NAME, &danhSach, &thang, &nam);
    if (ketQuaDoc == 1) {
        printf("Da doc %d nhan vien tu %s. Thang dang lam: %02d/%d.\n",
               demNhanVien(danhSach), FILE_NAME, thang, nam);
    } else if (ketQuaDoc == 0) {
        printf("Chua co file du lieu. Bat dau danh sach moi.\n");
        if (!docSoNguyen("Nhap thang cham cong (1-12): ", 1, 12, &thang)
            || !docSoNguyen("Nhap nam (2000-9999): ", 2000, 9999, &nam)) {
            giaiPhongDanhSach(danhSach);
            return 1;
        }
    } else {
        giaiPhongDanhSach(danhSach);
        return 1;
    }

    for (;;) {
        hienThiMenu(thang, nam);

        if (!docSoNguyen("Lua chon: ", 0, 11, &luaChon)) {
            break;
        }

        switch (luaChon) {
            case 1:
                themNhanVien(&danhSach);
                break;
            case 2:
                suaNhanVien(danhSach);
                break;
            case 3:
                xoaNhanVien(&danhSach);
                break;
            case 4:
                inDanhSach(danhSach);
                break;
            case 5:
                timVaInNhanVien(danhSach);
                break;
            case 6:
                nhapChamCong(danhSach);
                break;
            case 7:
                inBangLuong(danhSach, thang, nam);
                break;
            case 8:
                sapXepTheoLuong(danhSach);
                break;
            case 9:
                locTheoThucLinh(danhSach);
                break;
            case 10:
                doiThang(danhSach, &thang, &nam);
                break;
            case 11:
                luuDuLieu(FILE_NAME, danhSach, thang, nam);
                break;
            case 0:
                if (luuDuLieu(FILE_NAME, danhSach, thang, nam)) {
                    printf("Da luu du lieu. Tam biet!\n");
                } else {
                    printf("Luu du lieu that bai.\n");
                }
                giaiPhongDanhSach(danhSach);
                return 0;
        }
    }

    giaiPhongDanhSach(danhSach);
    return 0;
}
