# ภาค 21: ความสัมพันธ์ระหว่างตาราง (Table Relationships)

## ข้อมูลตัวอย่าง (Sample Data)

ก่อนเริ่มเรียน JOIN เราต้องมีข้อมูลตัวอย่างที่ใช้งานตลอดคอร์สนี้ มาสร้างฐานข้อมูลและใส่ข้อมูลกัน

```sql
-- ===================================================
-- สร้างโครงสร้างตาราง
-- ===================================================

CREATE TABLE departments (
    dept_id INT PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL,
    location VARCHAR(100),
    budget DECIMAL(15,2)
);

CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    dept_id INT REFERENCES departments(dept_id),
    salary DECIMAL(10,2),
    hire_date DATE,
    manager_id INT REFERENCES employees(emp_id),
    job_title VARCHAR(100)
);

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100) UNIQUE,
    city VARCHAR(50),
    country VARCHAR(50),
    created_at TIMESTAMP
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    category VARCHAR(50),
    price DECIMAL(10,2),
    stock_quantity INT,
    supplier_id INT
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    order_date DATE,
    total_amount DECIMAL(12,2),
    status VARCHAR(20),
    shipped_date DATE
);

CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT REFERENCES orders(order_id),
    product_id INT REFERENCES products(product_id),
    quantity INT,
    unit_price DECIMAL(10,2),
    discount DECIMAL(5,2)
);

-- ===================================================
-- ใส่ข้อมูล departments (10 แผนก)
-- ===================================================

INSERT INTO departments (dept_id, dept_name, location, budget) VALUES
(1,  'Engineering',       'Bangkok Floor 10',    5000000.00),
(2,  'Marketing',         'Bangkok Floor 8',     2000000.00),
(3,  'Sales',             'Bangkok Floor 7',     3000000.00),
(4,  'Human Resources',   'Bangkok Floor 6',     1500000.00),
(5,  'Finance',           'Bangkok Floor 9',     1800000.00),
(6,  'Operations',        'Chiang Mai Office',   2500000.00),
(7,  'Customer Support',  'Bangkok Floor 5',     1200000.00),
(8,  'Research & Dev',    'Bangkok Floor 11',    4000000.00),
(9,  'Legal',             'Bangkok Floor 4',      900000.00),
(10, 'Executive',         'Bangkok Floor 12',    8000000.00);

-- ===================================================
-- ใส่ข้อมูล employees (50 พนักงาน)
-- ===================================================

INSERT INTO employees (emp_id, first_name, last_name, email, dept_id, salary, hire_date, manager_id, job_title) VALUES
(1,  'Somchai',   'Jaidee',      'somchai.j@company.com',      10, 250000.00, '2015-01-15', NULL,  'CEO'),
(2,  'Wanchai',   'Thongdee',    'wanchai.t@company.com',      10, 200000.00, '2015-03-01', 1,    'CTO'),
(3,  'Pranee',    'Suksawat',    'pranee.s@company.com',       10, 195000.00, '2015-03-15', 1,    'CFO'),
(4,  'Narong',    'Charoenwong', 'narong.c@company.com',       1,  180000.00, '2016-01-10', 2,    'VP Engineering'),
(5,  'Siriporn',  'Meekhun',     'siriporn.m@company.com',     1,  150000.00, '2016-06-01', 4,    'Senior Engineer'),
(6,  'Kittipong', 'Wannaporn',   'kittipong.w@company.com',    1,  140000.00, '2017-02-15', 4,    'Senior Engineer'),
(7,  'Malee',     'Siriwan',     'malee.s@company.com',        1,  120000.00, '2018-05-01', 4,    'Engineer'),
(8,  'Thanapol',  'Jitporn',     'thanapol.j@company.com',     1,  115000.00, '2019-01-20', 4,    'Engineer'),
(9,  'Nopparat',  'Kaewkla',     'nopparat.k@company.com',     1,  110000.00, '2019-07-01', 4,    'Junior Engineer'),
(10, 'Suwan',     'Phongphan',   'suwan.p@company.com',        1,  105000.00, '2020-03-15', 4,    'Junior Engineer'),
(11, 'Apinya',    'Thongchan',   'apinya.t@company.com',       2,  160000.00, '2016-04-01', 1,    'VP Marketing'),
(12, 'Chatree',   'Wongsiri',    'chatree.w@company.com',      2,  130000.00, '2017-08-15', 11,   'Marketing Manager'),
(13, 'Duangjai',  'Saengthong',  'duangjai.s@company.com',     2,  100000.00, '2018-11-01', 11,   'Digital Marketer'),
(14, 'Ekachai',   'Limthong',    'ekachai.l@company.com',      2,   95000.00, '2019-03-01', 11,   'Content Writer'),
(15, 'Fon',       'Srirak',      'fon.s@company.com',          2,   85000.00, '2020-01-15', 11,   'Social Media'),
(16, 'Gritsana',  'Petcharat',   'gritsana.p@company.com',     3,  170000.00, '2016-02-01', 1,    'VP Sales'),
(17, 'Hansa',     'Khamthai',    'hansa.k@company.com',        3,  140000.00, '2017-05-15', 16,   'Sales Manager'),
(18, 'Ittipat',   'Wongwean',    'ittipat.w@company.com',      3,  120000.00, '2018-09-01', 16,   'Senior Sales'),
(19, 'Jintana',   'Phonsuk',     'jintana.p@company.com',      3,  110000.00, '2019-04-15', 16,   'Sales Rep'),
(20, 'Kamonwan',  'Rattana',     'kamonwan.r@company.com',     3,  100000.00, '2020-02-01', 16,   'Sales Rep'),
(21, 'Ladda',     'Thaworn',     'ladda.t@company.com',        4,  155000.00, '2016-07-01', 1,    'HR Director'),
(22, 'Montri',    'Suwannee',    'montri.s@company.com',       4,  125000.00, '2017-10-15', 21,   'HR Manager'),
(23, 'Natthida',  'Chaichana',   'natthida.c@company.com',     4,   95000.00, '2019-01-10', 21,   'HR Officer'),
(24, 'Oranuch',   'Kanjana',     'oranuch.k@company.com',      4,   88000.00, '2020-05-01', 21,   'Recruiter'),
(25, 'Panya',     'Moonwan',     'panya.m@company.com',        5,  165000.00, '2016-09-01', 1,    'Finance Director'),
(26, 'Rachen',    'Sombat',      'rachen.s@company.com',       5,  135000.00, '2017-12-01', 25,   'Finance Manager'),
(27, 'Saovapak',  'Thadee',      'saovapak.t@company.com',     5,  105000.00, '2019-02-15', 25,   'Accountant'),
(28, 'Teerasak',  'Phanthai',    'teerasak.p@company.com',     5,   95000.00, '2020-07-01', 25,   'Junior Accountant'),
(29, 'Udom',      'Wongthai',    'udom.w@company.com',         6,  158000.00, '2016-11-01', 1,    'Operations Director'),
(30, 'Vanida',    'Srichang',    'vanida.s@company.com',       6,  128000.00, '2018-01-15', 29,   'Operations Manager'),
(31, 'Wipa',      'Pakdee',      'wipa.p@company.com',         6,   98000.00, '2019-06-01', 29,   'Operations Officer'),
(32, 'Xutthana',  'Phromma',     'xutthana.p@company.com',     6,   88000.00, '2020-09-01', 29,   'Logistics Staff'),
(33, 'Yuphin',    'Kerdnoi',     'yuphin.k@company.com',       7,  145000.00, '2017-03-01', 1,    'Support Director'),
(34, 'Zara',      'Sirisak',     'zara.s@company.com',         7,  115000.00, '2018-07-15', 33,   'Support Manager'),
(35, 'Anan',      'Phirom',      'anan.p@company.com',         7,   85000.00, '2019-10-01', 33,   'Support Agent'),
(36, 'Banjong',   'Chanpen',     'banjong.c@company.com',      7,   80000.00, '2020-11-01', 33,   'Support Agent'),
(37, 'Chalida',   'Suwit',       'chalida.s@company.com',      8,  175000.00, '2016-05-01', 2,    'R&D Director'),
(38, 'Danai',     'Pattana',     'danai.p@company.com',        8,  155000.00, '2017-09-01', 37,   'Senior Researcher'),
(39, 'Emsawat',   'Kratip',      'emsawat.k@company.com',      8,  140000.00, '2018-03-15', 37,   'Researcher'),
(40, 'Faiw',      'Srisuwan',    'faiw.s@company.com',         8,  125000.00, '2019-08-01', 37,   'Junior Researcher'),
(41, 'Gawin',     'Thanakoon',   'gawin.t@company.com',        8,  115000.00, '2020-04-15', 37,   'Research Intern'),
(42, 'Hathai',    'Wongtong',    'hathai.w@company.com',       9,  162000.00, '2017-01-01', 1,    'Legal Director'),
(43, 'Isara',     'Chanok',      'isara.c@company.com',        9,  135000.00, '2018-05-15', 42,   'Senior Lawyer'),
(44, 'Jirat',     'Phakphoom',   'jirat.p@company.com',        9,  115000.00, '2019-11-01', 42,   'Lawyer'),
(45, 'Kamolwan',  'Siriphan',    'kamolwan.s@company.com',     9,   95000.00, '2020-08-15', 42,   'Legal Officer'),
(46, 'Kitisak',   'Wongkham',    'kitisak.w@company.com',      1,  132000.00, '2018-02-01', 4,    'DevOps Engineer'),
(47, 'Lalita',    'Phothong',    'lalita.p@company.com',       1,  128000.00, '2018-08-15', 4,    'QA Engineer'),
(48, 'Manit',     'Saetang',     'manit.s@company.com',        2,   92000.00, '2021-01-10', 11,   'Marketing Analyst'),
(49, 'Niran',     'Wattana',     'niran.w@company.com',        3,  108000.00, '2021-03-15', 16,   'Sales Analyst'),
(50, 'Orawan',    'Phuttha',     'orawan.p@company.com',       5,   90000.00, '2021-06-01', 25,   'Financial Analyst');

-- ===================================================
-- ใส่ข้อมูล customers (60 ลูกค้า)
-- ===================================================

INSERT INTO customers (customer_id, first_name, last_name, email, city, country, created_at) VALUES
(1,  'Arisa',     'Wannapat',   'arisa.w@email.com',     'Bangkok',       'Thailand',    '2022-01-15 10:30:00'),
(2,  'Boonchai',  'Srinak',     'boonchai.s@email.com',  'Chiang Mai',    'Thailand',    '2022-02-20 14:15:00'),
(3,  'Chanya',    'Phonsuk',    'chanya.p@email.com',    'Phuket',        'Thailand',    '2022-03-10 09:00:00'),
(4,  'Damrong',   'Thongdee',   'damrong.t@email.com',   'Bangkok',       'Thailand',    '2022-03-25 11:45:00'),
(5,  'Ekapon',    'Jaidee',     'ekapon.j@email.com',    'Nonthaburi',    'Thailand',    '2022-04-05 16:20:00'),
(6,  'Fanta',     'Moonwong',   'fanta.m@email.com',     'Pattaya',       'Thailand',    '2022-04-18 08:30:00'),
(7,  'Gorn',      'Suwannak',   'gorn.s@email.com',      'Hat Yai',       'Thailand',    '2022-05-02 13:00:00'),
(8,  'Hathai',    'Kanchai',    'hathai.k@email.com',    'Bangkok',       'Thailand',    '2022-05-20 17:30:00'),
(9,  'Ittiphat',  'Prathom',    'ittiphat.p@email.com',  'Samut Prakan',  'Thailand',    '2022-06-08 10:15:00'),
(10, 'Janya',     'Charoenpak', 'janya.c@email.com',     'Khon Kaen',     'Thailand',    '2022-06-25 14:45:00'),
(11, 'Kongkiat',  'Wongsiri',   'kongkiat.w@email.com',  'Udon Thani',    'Thailand',    '2022-07-12 09:30:00'),
(12, 'Lalida',    'Somboon',    'lalida.s@email.com',    'Nakhon Ratchasima', 'Thailand', '2022-07-28 15:00:00'),
(13, 'Manida',    'Phanthai',   'manida.p@email.com',    'Bangkok',       'Thailand',    '2022-08-14 11:20:00'),
(14, 'Narit',     'Thawee',     'narit.t@email.com',     'Lampang',       'Thailand',    '2022-08-30 16:45:00'),
(15, 'Ornuma',    'Sirisak',    'ornuma.s@email.com',    'Chon Buri',     'Thailand',    '2022-09-15 08:00:00'),
(16, 'Pichet',    'Khamthai',   'pichet.k@email.com',    'Bangkok',       'Thailand',    '2022-09-28 13:30:00'),
(17, 'Ratchada',  'Phetprakorn', 'ratchada.p@email.com', 'Phuket',        'Thailand',    '2022-10-10 10:00:00'),
(18, 'Saran',     'Mongkol',    'saran.m@email.com',     'Chiang Rai',    'Thailand',    '2022-10-25 15:15:00'),
(19, 'Tanya',     'Wattana',    'tanya.w@email.com',     'Bangkok',       'Thailand',    '2022-11-08 09:45:00'),
(20, 'Ubon',      'Saetang',    'ubon.s@email.com',      'Nakhon Si Thammarat', 'Thailand', '2022-11-20 14:00:00'),
(21, 'Varich',    'Phiromsak',  'varich.p@email.com',    'Bangkok',       'Thailand',    '2022-12-05 11:00:00'),
(22, 'Wipada',    'Srichan',    'wipada.s@email.com',    'Pathum Thani',  'Thailand',    '2022-12-18 16:30:00'),
(23, 'Yada',      'Kerdnoi',    'yada.k@email.com',      'Bangkok',       'Thailand',    '2023-01-10 08:15:00'),
(24, 'Zina',      'Phomchan',   'zina.p@email.com',      'Samut Sakhon',  'Thailand',    '2023-01-25 13:45:00'),
(25, 'Anong',     'Rattana',    'anong.r@email.com',     'Bangkok',       'Thailand',    '2023-02-08 10:30:00'),
(26, 'Benjamas',  'Thipwan',    'benjamas.t@email.com',  'Kanchanaburi',  'Thailand',    '2023-02-22 15:00:00'),
(27, 'Chadarat',  'Wongthai',   'chadarat.w@email.com',  'Nonthaburi',    'Thailand',    '2023-03-07 09:00:00'),
(28, 'Decha',     'Phromma',    'decha.p@email.com',     'Bangkok',       'Thailand',    '2023-03-20 14:20:00'),
(29, 'Ekapol',    'Srisuk',     'ekapol.s@email.com',    'Pattaya',       'Thailand',    '2023-04-04 11:45:00'),
(30, 'Fara',      'Janpeng',    'fara.j@email.com',      'Hua Hin',       'Thailand',    '2023-04-18 16:10:00'),
(31, 'Garuda',    'Prachak',    'garuda.p@email.com',    'Bangkok',       'Thailand',    '2023-05-02 08:30:00'),
(32, 'Hom',       'Saengsawat', 'hom.s@email.com',       'Chiang Mai',    'Thailand',    '2023-05-15 13:00:00'),
(33, 'Irin',      'Kittiwong',  'irin.k@email.com',      'Bangkok',       'Thailand',    '2023-05-28 17:15:00'),
(34, 'Jariya',    'Phon',       'jariya.p@email.com',    'Ayutthaya',     'Thailand',    '2023-06-12 10:00:00'),
(35, 'Kannika',   'Thaweesak',  'kannika.t@email.com',   'Bangkok',       'Thailand',    '2023-06-25 14:30:00'),
(36, 'Lawan',     'Sirisuwan',  'lawan.s@email.com',     'Phuket',        'Thailand',    '2023-07-08 09:15:00'),
(37, 'Monthon',   'Wongpan',    'monthon.w@email.com',   'Chon Buri',     'Thailand',    '2023-07-22 15:45:00'),
(38, 'Nisa',      'Phanchan',   'nisa.p@email.com',      'Bangkok',       'Thailand',    '2023-08-05 11:30:00'),
(39, 'Orathai',   'Kumlung',    'orathai.k@email.com',   'Nakhon Ratchasima', 'Thailand', '2023-08-18 16:00:00'),
(40, 'Pimchanok', 'Siriphan',   'pimchanok.s@email.com', 'Bangkok',       'Thailand',    '2023-09-01 08:45:00'),
(41, 'Rachata',   'Thong',      'rachata.t@email.com',   'Hat Yai',       'Thailand',    '2023-09-14 13:15:00'),
(42, 'Saranya',   'Prathum',    'saranya.p@email.com',   'Bangkok',       'Thailand',    '2023-09-27 17:30:00'),
(43, 'Tawan',     'Moonkham',   'tawan.m@email.com',     'Ubon Ratchathani', 'Thailand',  '2023-10-11 10:15:00'),
(44, 'Usanee',    'Wattanakul', 'usanee.w@email.com',    'Bangkok',       'Thailand',    '2023-10-24 14:45:00'),
(45, 'Vimol',     'Srimongkol', 'vimol.s@email.com',     'Rayong',        'Thailand',    '2023-11-07 09:30:00'),
(46, 'Wanvisa',   'Chanpheng',  'wanvisa.c@email.com',   'Bangkok',       'Thailand',    '2023-11-20 15:00:00'),
(47, 'Xianlin',   'Tanakorn',   'xianlin.t@email.com',   'Bangkok',       'Thailand',    '2023-12-04 10:45:00'),
(48, 'Yaowapa',   'Srithai',    'yaowapa.s@email.com',   'Chiang Mai',    'Thailand',    '2023-12-17 16:15:00'),
(49, 'Zulaikha',  'Phakphon',   'zulaikha.p@email.com',  'Pattani',       'Thailand',    '2024-01-08 08:00:00'),
(50, 'Arunee',    'Thongdee',   'arunee.t@email.com',    'Bangkok',       'Thailand',    '2024-01-22 13:30:00'),
(51, 'Burin',     'Suksorn',    'burin.s@email.com',     'Samut Prakan',  'Thailand',    '2024-02-05 17:00:00'),
(52, 'Chanoknan', 'Phaiboon',   'chanoknan.p@email.com', 'Bangkok',       'Thailand',    '2024-02-19 11:15:00'),
(53, 'Dawan',     'Intharit',   'dawan.i@email.com',     'Khon Kaen',     'Thailand',    '2024-03-04 09:45:00'),
(54, 'Ekthai',    'Chalermsuk', 'ekthai.c@email.com',    'Bangkok',       'Thailand',    '2024-03-18 14:00:00'),
(55, 'Fuang',     'Namwong',    'fuang.n@email.com',     'Nonthaburi',    'Thailand',    '2024-04-01 16:30:00'),
(56, 'Ganlaya',   'Phromwan',   'ganlaya.p@email.com',   'Bangkok',       'Thailand',    '2024-04-15 08:30:00'),
(57, 'Hongsa',    'Suwan',      'hongsa.s@email.com',    'Chiang Mai',    'Thailand',    '2024-04-29 13:00:00'),
(58, 'Ingkarat',  'Makhun',     'ingkarat.m@email.com',  'Phuket',        'Thailand',    '2024-05-13 17:15:00'),
(59, 'Jaruk',     'Phetchara',  'jaruk.p@email.com',     'Bangkok',       'Thailand',    '2024-05-27 10:30:00'),
(60, 'Kanya',     'Wichai',     'kanya.w@email.com',     'Hat Yai',       'Thailand',    '2024-06-10 15:45:00');

-- ===================================================
-- ใส่ข้อมูล products (50 สินค้า)
-- ===================================================

INSERT INTO products (product_id, product_name, category, price, stock_quantity, supplier_id) VALUES
(1,  'Laptop Pro 15',          'Electronics',   45000.00, 50,  101),
(2,  'Laptop Air 13',          'Electronics',   32000.00, 75,  101),
(3,  'Desktop PC Core i7',     'Electronics',   28000.00, 40,  101),
(4,  'Monitor 27 inch 4K',     'Electronics',   15000.00, 60,  102),
(5,  'Wireless Keyboard',      'Electronics',    1800.00, 200, 102),
(6,  'Wireless Mouse',         'Electronics',    1200.00, 250, 102),
(7,  'USB-C Hub 7-in-1',       'Electronics',    2500.00, 150, 102),
(8,  'Webcam HD 1080p',        'Electronics',    3500.00, 100, 103),
(9,  'Headphones ANC',         'Electronics',    8500.00, 80,  103),
(10, 'Smartphone Galaxy S24',  'Electronics',   35000.00, 120, 101),
(11, 'Tablet iPad Pro',        'Electronics',   42000.00, 60,  101),
(12, 'Smart Watch Series 9',   'Electronics',   18000.00, 90,  103),
(13, 'Bluetooth Speaker',      'Electronics',    4500.00, 180, 103),
(14, 'Power Bank 20000mAh',    'Electronics',    2200.00, 300, 102),
(15, 'Laptop Stand Aluminum',  'Accessories',   2800.00, 120, 104),
(16, 'Office Chair Ergonomic', 'Furniture',     12000.00, 30,  105),
(17, 'Standing Desk Electric', 'Furniture',     25000.00, 20,  105),
(18, 'Bookshelf 5 Tier',       'Furniture',      4500.00, 45,  105),
(19, 'Desk Lamp LED',          'Accessories',   1500.00, 200, 104),
(20, 'Whiteboard Magnetic',    'Office',         3800.00, 60,  106),
(21, 'Sticky Notes Pack',      'Stationery',      350.00, 500, 106),
(22, 'Ballpoint Pens 12pc',    'Stationery',      280.00, 800, 106),
(23, 'Notebook A4 Pack',       'Stationery',      450.00, 600, 106),
(24, 'Paper Clips Box',        'Stationery',       90.00, 1000,106),
(25, 'Stapler Heavy Duty',     'Office',          650.00, 150, 106),
(26, 'File Cabinet 4 Drawer',  'Furniture',      8500.00, 25,  105),
(27, 'Printer Laser Color',    'Electronics',   18500.00, 35,  101),
(28, 'Scanner Document',       'Electronics',    9500.00, 40,  101),
(29, 'Projector 4K',           'Electronics',   35000.00, 20,  101),
(30, 'HDMI Cable 2m',          'Accessories',    380.00, 500, 102),
(31, 'Extension Cord 5m',      'Accessories',    550.00, 300, 102),
(32, 'Cable Management Box',   'Accessories',    890.00, 200, 104),
(33, 'Monitor Arm Dual',       'Accessories',   3500.00, 80,  104),
(34, 'Keyboard Wrist Rest',    'Accessories',    680.00, 250, 104),
(35, 'Mouse Pad XL',           'Accessories',    450.00, 400, 104),
(36, 'Coffee Machine Auto',    'Appliances',    12000.00, 25,  107),
(37, 'Water Dispenser Hot',    'Appliances',     8500.00, 30,  107),
(38, 'Microwave Oven',         'Appliances',     4500.00, 20,  107),
(39, 'Air Purifier HEPA',      'Appliances',     9800.00, 35,  107),
(40, 'Portable AC 12000BTU',   'Appliances',    22000.00, 15,  107),
(41, 'Security Camera Set',    'Security',       8500.00, 50,  108),
(42, 'Access Control System',  'Security',      25000.00, 10,  108),
(43, 'Fire Extinguisher',      'Safety',         1800.00, 100, 109),
(44, 'First Aid Kit',          'Safety',          950.00, 150, 109),
(45, 'Hard Hat Safety',        'Safety',          680.00, 200, 109),
(46, 'Software MS Office',     'Software',       4500.00, 999, 110),
(47, 'Antivirus 1yr License',  'Software',       1800.00, 999, 110),
(48, 'Cloud Storage 1TB',      'Software',       1200.00, 999, 110),
(49, 'VPN Service Annual',     'Software',       2400.00, 999, 110),
(50, 'Project Mgmt Software',  'Software',       3600.00, 999, 110);

-- ===================================================
-- ใส่ข้อมูล orders (70 ออร์เดอร์)
-- ===================================================

INSERT INTO orders (order_id, customer_id, order_date, total_amount, status, shipped_date) VALUES
(1,   1,  '2024-01-05', 47800.00,  'completed',  '2024-01-07'),
(2,   2,  '2024-01-08', 3700.00,   'completed',  '2024-01-10'),
(3,   3,  '2024-01-12', 13500.00,  'completed',  '2024-01-14'),
(4,   4,  '2024-01-15', 2980.00,   'completed',  '2024-01-17'),
(5,   5,  '2024-01-18', 35000.00,  'completed',  '2024-01-20'),
(6,   6,  '2024-01-22', 8930.00,   'completed',  '2024-01-24'),
(7,   7,  '2024-01-25', 1580.00,   'completed',  '2024-01-27'),
(8,   8,  '2024-02-01', 42000.00,  'completed',  '2024-02-03'),
(9,   9,  '2024-02-05', 6700.00,   'completed',  '2024-02-07'),
(10,  10, '2024-02-08', 22000.00,  'completed',  '2024-02-10'),
(11,  11, '2024-02-12', 4500.00,   'completed',  '2024-02-14'),
(12,  12, '2024-02-15', 18200.00,  'completed',  '2024-02-17'),
(13,  13, '2024-02-19', 9800.00,   'completed',  '2024-02-21'),
(14,  14, '2024-02-22', 3150.00,   'completed',  '2024-02-24'),
(15,  15, '2024-02-26', 15000.00,  'completed',  '2024-02-28'),
(16,  16, '2024-03-01', 28000.00,  'completed',  '2024-03-03'),
(17,  17, '2024-03-05', 7200.00,   'completed',  '2024-03-07'),
(18,  18, '2024-03-08', 45500.00,  'completed',  '2024-03-10'),
(19,  19, '2024-03-12', 2700.00,   'completed',  '2024-03-14'),
(20,  20, '2024-03-15', 12000.00,  'completed',  '2024-03-17'),
(21,  21, '2024-03-19', 35500.00,  'completed',  '2024-03-21'),
(22,  22, '2024-03-22', 5600.00,   'completed',  '2024-03-24'),
(23,  23, '2024-03-26', 8900.00,   'completed',  '2024-03-28'),
(24,  24, '2024-04-01', 19800.00,  'completed',  '2024-04-03'),
(25,  25, '2024-04-04', 3400.00,   'completed',  '2024-04-06'),
(26,  26, '2024-04-08', 25000.00,  'completed',  '2024-04-10'),
(27,  27, '2024-04-11', 6800.00,   'completed',  '2024-04-13'),
(28,  28, '2024-04-15', 14500.00,  'completed',  '2024-04-17'),
(29,  29, '2024-04-18', 42000.00,  'completed',  '2024-04-20'),
(30,  30, '2024-04-22', 9200.00,   'completed',  '2024-04-24'),
(31,  1,  '2024-04-25', 18000.00,  'completed',  '2024-04-27'),
(32,  2,  '2024-05-01', 5500.00,   'completed',  '2024-05-03'),
(33,  3,  '2024-05-05', 22500.00,  'completed',  '2024-05-07'),
(34,  5,  '2024-05-08', 8100.00,   'completed',  '2024-05-10'),
(35,  8,  '2024-05-12', 35000.00,  'completed',  '2024-05-14'),
(36,  10, '2024-05-15', 4700.00,   'completed',  '2024-05-17'),
(37,  13, '2024-05-19', 16500.00,  'completed',  '2024-05-21'),
(38,  16, '2024-05-22', 7300.00,   'completed',  '2024-05-24'),
(39,  19, '2024-05-26', 28000.00,  'completed',  '2024-05-28'),
(40,  21, '2024-05-29', 9500.00,   'completed',  '2024-05-31'),
(41,  31, '2024-06-02', 15000.00,  'completed',  '2024-06-04'),
(42,  32, '2024-06-05', 42500.00,  'completed',  '2024-06-07'),
(43,  33, '2024-06-09', 6200.00,   'completed',  '2024-06-11'),
(44,  34, '2024-06-12', 19000.00,  'completed',  '2024-06-14'),
(45,  35, '2024-06-16', 3800.00,   'completed',  '2024-06-18'),
(46,  36, '2024-06-19', 8600.00,   'shipped',    '2024-06-21'),
(47,  37, '2024-06-23', 25500.00,  'shipped',    '2024-06-25'),
(48,  38, '2024-06-26', 11200.00,  'shipped',    '2024-06-28'),
(49,  39, '2024-07-01', 4900.00,   'processing', NULL),
(50,  40, '2024-07-04', 37000.00,  'processing', NULL),
(51,  41, '2024-07-08', 7800.00,   'processing', NULL),
(52,  42, '2024-07-11', 16800.00,  'processing', NULL),
(53,  43, '2024-07-15', 29500.00,  'pending',    NULL),
(54,  44, '2024-07-18', 5200.00,   'pending',    NULL),
(55,  45, '2024-07-22', 12400.00,  'pending',    NULL),
(56,  46, '2024-07-25', 43000.00,  'pending',    NULL),
(57,  47, '2024-07-29', 8700.00,   'cancelled',  NULL),
(58,  48, '2024-08-01', 21000.00,  'cancelled',  NULL),
(59,  1,  '2024-08-05', 15500.00,  'completed',  '2024-08-07'),
(60,  5,  '2024-08-08', 9800.00,   'completed',  '2024-08-10'),
(61,  10, '2024-08-12', 32000.00,  'completed',  '2024-08-14'),
(62,  15, '2024-08-15', 6400.00,   'completed',  '2024-08-17'),
(63,  20, '2024-08-19', 18700.00,  'completed',  '2024-08-21'),
(64,  25, '2024-08-22', 4100.00,   'completed',  '2024-08-24'),
(65,  30, '2024-08-26', 27800.00,  'shipped',    '2024-08-28'),
(66,  35, '2024-08-29', 8300.00,   'shipped',    NULL),
(67,  40, '2024-09-02', 14200.00,  'processing', NULL),
(68,  45, '2024-09-05', 38500.00,  'processing', NULL),
(69,  50, '2024-09-09', 6900.00,   'pending',    NULL),
(70,  55, '2024-09-12', 22000.00,  'pending',    NULL);

-- ===================================================
-- ใส่ข้อมูล order_items (100+ รายการ)
-- ===================================================

INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, discount) VALUES
(1,   1,  1,  1, 45000.00, 500.00),
(2,   1,  5,  2, 1800.00,   50.00),
(3,   2,  13, 1, 4500.00,  400.00),
(4,   2,  6,  1, 1200.00,    0.00),
(5,   3,  4,  1, 15000.00, 1000.00),
(6,   3,  19, 2, 1500.00,    0.00),
(7,   4,  5,  1, 1800.00,   50.00),
(8,   4,  6,  1, 1200.00,    0.00),
(9,   5,  10, 1, 35000.00,   0.00),
(10,  6,  9,  1, 8500.00,  200.00),
(11,  6,  7,  1, 2500.00,    0.00),
(12,  7,  22, 2,  280.00,    0.00),
(13,  7,  23, 2,  450.00,    0.00),
(14,  8,  11, 1, 42000.00,   0.00),
(15,  9,  8,  1, 3500.00,    0.00),
(16,  9,  7,  1, 2500.00,    0.00),
(17,  10, 17, 1, 25000.00,   0.00),
(18,  10, 19, 2, 1500.00,    0.00),
(19,  11, 13, 1, 4500.00,    0.00),
(20,  12, 27, 1, 18500.00, 500.00),
(21,  13, 39, 1, 9800.00,    0.00),
(22,  14, 25, 1,  650.00,    0.00),
(23,  14, 21, 5,  350.00,    0.00),
(24,  14, 22, 3,  280.00,    0.00),
(25,  15, 36, 1, 12000.00,   0.00),
(26,  15, 46, 1, 4500.00,    0.00),
(27,  16, 3,  1, 28000.00,   0.00),
(28,  17, 12, 1, 18000.00, 1000.00),
(29,  17, 30, 2,  380.00,    0.00),
(30,  18, 1,  1, 45000.00,   0.00),
(31,  18, 2,  1, 32000.00, 1000.00),
(32,  18, 5,  1, 1800.00,    0.00),
(33,  19, 22, 5,  280.00,    0.00),
(34,  19, 21, 3,  350.00,    0.00),
(35,  20, 36, 1, 12000.00,   0.00),
(36,  21, 10, 1, 35000.00,   0.00),
(37,  21, 12, 1, 18000.00, 1000.00),
(38,  22, 9,  1, 8500.00,    0.00),
(39,  22, 34, 2,  680.00,    0.00),
(40,  23, 39, 1, 9800.00,    0.00),
(41,  24, 4,  1, 15000.00,   0.00),
(42,  24, 33, 1, 3500.00,    0.00),
(43,  24, 35, 2,  450.00,    0.00),
(44,  25, 22, 5,  280.00,    0.00),
(45,  25, 23, 3,  450.00,    0.00),
(46,  26, 16, 2, 12000.00,   0.00),
(47,  27, 8,  1, 3500.00,    0.00),
(48,  27, 7,  1, 2500.00,    0.00),
(49,  27, 30, 2,  380.00,    0.00),
(50,  28, 28, 1, 9500.00,    0.00),
(51,  28, 8,  1, 3500.00,    0.00),
(52,  28, 7,  1, 2500.00,    0.00),
(53,  29, 11, 1, 42000.00,   0.00),
(54,  30, 9,  1, 8500.00,    0.00),
(55,  30, 19, 1, 1500.00,    0.00),
(56,  31, 16, 1, 12000.00,   0.00),
(57,  31, 17, 1, 25000.00, 1000.00),
(58,  32, 13, 1, 4500.00,    0.00),
(59,  32, 6,  1, 1200.00,    0.00),
(60,  33, 1,  1, 45000.00, 1000.00),
(61,  34, 9,  1, 8500.00,    0.00),
(62,  35, 10, 1, 35000.00,   0.00),
(63,  36, 5,  2, 1800.00,    0.00),
(64,  36, 6,  1, 1200.00,    0.00),
(65,  37, 4,  1, 15000.00,   0.00),
(66,  37, 7,  1, 2500.00,    0.00),
(67,  38, 12, 1, 18000.00, 1000.00),
(68,  39, 29, 1, 35000.00, 1000.00),
(69,  40, 9,  1, 8500.00,    0.00),
(70,  40, 19, 1, 1500.00,    0.00),
(71,  41, 36, 1, 12000.00,   0.00),
(72,  42, 1,  1, 45000.00, 1000.00),
(73,  43, 8,  1, 3500.00,    0.00),
(74,  43, 30, 5,  380.00,    0.00),
(75,  44, 4,  1, 15000.00,   0.00),
(76,  44, 16, 1, 12000.00, 1000.00),
(77,  45, 21, 5,  350.00,    0.00),
(78,  45, 22, 5,  280.00,    0.00),
(79,  46, 9,  1, 8500.00,    0.00),
(80,  46, 14, 1, 2200.00,    0.00),
(81,  47, 3,  1, 28000.00, 1000.00),
(82,  48, 28, 1, 9500.00,    0.00),
(83,  48, 8,  1, 3500.00,    0.00),
(84,  49, 5,  2, 1800.00,    0.00),
(85,  49, 6,  1, 1200.00,    0.00),
(86,  50, 10, 1, 35000.00,   0.00),
(87,  51, 12, 1, 18000.00,   0.00),
(88,  52, 4,  1, 15000.00,   0.00),
(89,  52, 7,  1, 2500.00,    0.00),
(90,  53, 29, 1, 35000.00, 1000.00),
(91,  54, 25, 2,  650.00,    0.00),
(92,  54, 22, 5,  280.00,    0.00),
(93,  55, 36, 1, 12000.00,   0.00),
(94,  56, 1,  1, 45000.00, 1000.00),
(95,  59, 2,  1, 32000.00, 1000.00),
(96,  60, 9,  1, 8500.00,    0.00),
(97,  61, 10, 1, 35000.00, 1000.00),
(98,  62, 7,  1, 2500.00,    0.00),
(99,  63, 16, 1, 12000.00,   0.00),
(100, 64, 22, 5,  280.00,    0.00),
(101, 65, 3,  1, 28000.00,   0.00),
(102, 66, 9,  1, 8500.00,    0.00),
(103, 67, 4,  1, 15000.00,   0.00),
(104, 68, 1,  1, 45000.00, 1000.00),
(105, 69, 12, 1, 18000.00, 1000.00),
(106, 70, 16, 2, 12000.00, 1000.00);
```

---

## บทที่ 21: ความสัมพันธ์ระหว่างตาราง (Table Relationships)

### ทำไมต้องเข้าใจความสัมพันธ์?

ก่อนที่เราจะเรียน JOIN ซึ่งเป็นหัวใจของการเขียน SQL ขั้นสูง เราต้องเข้าใจพื้นฐานสำคัญก่อนว่า **ทำไมถึงมีหลายตาราง** และ **ตารางเหล่านั้นเชื่อมโยงกันอย่างไร**

ฐานข้อมูลเชิงสัมพันธ์ (Relational Database) ถูกออกแบบมาเพื่อ:
- **ลดความซ้ำซ้อน** (Reduce Redundancy) ของข้อมูล
- **รักษาความถูกต้อง** (Maintain Integrity) ของข้อมูล
- **ประหยัดพื้นที่** (Save Storage) จัดเก็บข้อมูล
- **ง่ายต่อการแก้ไข** (Easy Maintenance) ข้อมูล

### แนวคิด Normalization

**การ Normalize** คือการแยกข้อมูลออกเป็นตารางต่างๆ เพื่อลดความซ้ำซ้อน

```
ตัวอย่างข้อมูลที่ไม่ได้ Normalize (BAD):
┌─────────────────────────────────────────────────────────────────┐
│ order_id │ customer_name │ customer_email    │ product_name  │  │
├──────────┼───────────────┼───────────────────┼───────────────┤  │
│ 1        │ Somchai Jaidee│ somchai@email.com │ Laptop Pro 15 │  │
│ 2        │ Somchai Jaidee│ somchai@email.com │ Mouse         │  │  ← ซ้ำ!
│ 3        │ Wanchai Thong │ wanchai@email.com │ Keyboard      │  │
└─────────────────────────────────────────────────────────────────┘

ปัญหา:
- ถ้า Somchai เปลี่ยน email ต้องแก้หลายแถว
- ข้อมูลซ้ำกันมาก ทำให้ตารางใหญ่
- อาจเกิด Anomaly ข้อมูลไม่ตรงกัน
```

```
ข้อมูลที่ Normalize แล้ว (GOOD):
customers:                    orders:
┌─────┬────────┬───────────┐  ┌──────────┬─────────────┐
│ id  │ name   │ email     │  │ order_id │ customer_id │
├─────┼────────┼───────────┤  ├──────────┼─────────────┤
│ 1   │ Somchai│ somchai@  │  │ 1        │ 1           │
│ 2   │ Wanchai│ wanchai@  │  │ 2        │ 1           │
└─────┴────────┴───────────┘  │ 3        │ 2           │
                               └──────────┴─────────────┘
ดีกว่าเพราะ: แก้ email ครั้งเดียว มีผลกับทุก order
```

---

## ประเภทของความสัมพันธ์ (Types of Relationships)

### 1. ความสัมพันธ์แบบ One-to-One (1:1)

**ความหมาย:** หนึ่งระเบียนในตาราง A สัมพันธ์กับหนึ่งระเบียนในตาราง B เท่านั้น

**ตัวอย่างจริง:**
- พนักงาน 1 คน มีรหัสพนักงาน 1 รหัส
- ผู้ใช้ 1 คน มี profile 1 โปรไฟล์
- ประเทศ 1 ประเทศ มีเมืองหลวง 1 เมือง

```
แผนภาพ One-to-One:

employees          employee_profiles
┌─────────┐        ┌──────────────────┐
│ emp_id  │────────│ emp_id (FK)      │
│ name    │   1:1  │ bio              │
│ ...     │        │ photo_url        │
└─────────┘        └──────────────────┘

ตัวอย่าง SQL:
```

```sql
-- ตัวอย่าง One-to-One: employees กับ employee_profiles
-- (สมมติมีตาราง employee_profiles)
CREATE TABLE employee_profiles (
    profile_id INT PRIMARY KEY,
    emp_id INT UNIQUE REFERENCES employees(emp_id),  -- UNIQUE ทำให้เป็น 1:1
    bio TEXT,
    photo_url VARCHAR(200),
    linkedin_url VARCHAR(200)
);

-- แต่ละพนักงานมีได้แค่ 1 profile (UNIQUE constraint)
INSERT INTO employee_profiles VALUES
(1, 1, 'CEO with 20 years experience', '/photos/emp1.jpg', NULL),
(2, 2, 'Tech leader, loves open source', '/photos/emp2.jpg', 'linkedin.com/wanchai');
```

**เมื่อใดควรใช้ One-to-One:**
- เมื่อข้อมูลบางส่วนไม่จำเป็นต้องใช้บ่อย (ประหยัด I/O)
- เมื่อต้องการแยก sensitive data (เช่น เงินเดือน แยกจากข้อมูลทั่วไป)
- เมื่อต้องการ optional relationship

---

### 2. ความสัมพันธ์แบบ One-to-Many (1:N) — พบมากที่สุด!

**ความหมาย:** หนึ่งระเบียนในตาราง A สัมพันธ์กับหลายระเบียนในตาราง B

**ตัวอย่างจริงในฐานข้อมูลของเรา:**
- 1 department → มีพนักงานได้หลายคน
- 1 customer → สั่งซื้อได้หลาย order
- 1 order → มีสินค้าได้หลาย order_items

```
แผนภาพ One-to-Many:

departments              employees
┌──────────┐             ┌─────────────┐
│ dept_id  │◄────────────│ dept_id(FK) │
│ dept_name│   1 : Many  │ emp_id      │
│ ...      │             │ first_name  │
└──────────┘             │ ...         │
     1                   └─────────────┘
                              N
หมายความว่า:
- dept_id = 1 (Engineering) มีพนักงาน 12 คน
- dept_id = 2 (Marketing)   มีพนักงาน  5 คน
- และอื่นๆ
```

```sql
-- ตรวจสอบความสัมพันธ์ 1:N ระหว่าง departments และ employees
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS employee_count
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name
ORDER BY employee_count DESC;

-- ผลลัพธ์:
-- Engineering    : 12 คน  (dept_id 1)
-- Sales          :  5 คน  (dept_id 3)
-- Marketing      :  5 คน  (dept_id 2)
-- ...
```

**ลักษณะของ One-to-Many:**
- ตาราง "หนึ่ง" = Parent Table (departments, customers)
- ตาราง "หลาย" = Child Table (employees, orders)
- Foreign Key อยู่ที่ Child Table เสมอ!

```
Foreign Key อยู่ที่ไหน?

departments          employees
┌──────────┐         ┌─────────────┐
│ dept_id  │         │ emp_id      │
│ PK       │◄────────│ dept_id     │ ← FK อยู่ที่นี่!
└──────────┘         │ ...         │
                     └─────────────┘
                     (Child Table)
```

---

### 3. ความสัมพันธ์แบบ Many-to-Many (M:N)

**ความหมาย:** หลายระเบียนในตาราง A สัมพันธ์กับหลายระเบียนในตาราง B

**ตัวอย่างจริง:**
- นักศึกษา M คน เรียนได้หลาย N วิชา, แต่ 1 วิชาก็มีได้หลายนักศึกษา
- 1 order มีได้หลาย products, และ 1 product ก็อยู่ในได้หลาย orders
- พนักงาน M คน รับผิดชอบได้หลาย N โปรเจกต์

**วิธีแก้ปัญหา:** ใช้ **Junction Table** (ตารางเชื่อม) หรือ Bridge Table

```
แผนภาพ Many-to-Many:

orders           order_items           products
┌──────────┐    ┌─────────────────┐    ┌─────────────┐
│ order_id │◄───│ order_id  (FK)  │    │ product_id  │
│ customer │    │ product_id (FK) │───►│ product_name│
│ ...      │    │ quantity        │    │ price       │
└──────────┘    │ unit_price      │    │ ...         │
                │ discount        │    └─────────────┘
                └─────────────────┘
                  (Junction Table)

order_items เป็น Junction Table ที่:
- มี FK ชี้ไป orders (order_id)
- มี FK ชี้ไป products (product_id)
- มีข้อมูลเพิ่มเติม (quantity, unit_price, discount)
```

```sql
-- ตัวอย่าง Many-to-Many: หา products ทั้งหมดใน order 18
SELECT 
    o.order_id,
    p.product_name,
    oi.quantity,
    oi.unit_price
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.order_id = 18;

-- ผลลัพธ์:
-- order 18 มีหลาย products (Laptop Pro 15, Laptop Air 13, Wireless Keyboard)
```

---

## Entity-Relationship Diagram (ERD) ของฐานข้อมูลตัวอย่าง

```
ER Diagram แบบ ASCII Art

                    ┌───────────────────┐
                    │    departments    │
                    │─────────────────  │
                    │ dept_id    PK     │
                    │ dept_name  NOT NULL│
                    │ location          │
                    │ budget            │
                    └────────┬──────────┘
                             │ 1
                             │ has many
                             │ N
                    ┌────────▼──────────┐
                    │     employees     │
                    │─────────────────  │
                    │ emp_id     PK     │
                    │ first_name NOT NULL│
                    │ last_name  NOT NULL│
                    │ email      UNIQUE │
              ┌─────│ dept_id    FK     │
              │     │ salary            │
              │     │ hire_date         │
              └────►│ manager_id FK(self)│  ← Self-referential!
                    │ job_title         │
                    └───────────────────┘


  customers                              orders                         order_items                     products
┌───────────────┐                  ┌───────────────┐               ┌───────────────────┐          ┌───────────────┐
│ customer_id PK│                  │ order_id   PK │               │ item_id        PK │          │ product_id PK │
│ first_name    │                  │ customer_id FK│               │ order_id       FK │          │ product_name  │
│ last_name     │──────────────────│ order_date    │───────────────│ product_id     FK │──────────│ category      │
│ email  UNIQUE │   1 : Many       │ total_amount  │  1 : Many     │ quantity          │ Many : 1 │ price         │
│ city          │                  │ status        │               │ unit_price        │          │ stock_quantity│
│ country       │                  │ shipped_date  │               │ discount          │          │ supplier_id   │
│ created_at    │                  └───────────────┘               └───────────────────┘          └───────────────┘
└───────────────┘

                                    Relationship Summary:
                                    ─────────────────────
                                    departments  1──N  employees
                                    employees    1──N  employees (self-ref: manager)
                                    customers    1──N  orders
                                    orders       1──N  order_items
                                    products     1──N  order_items
                                    orders      M──N   products  (via order_items)
```

---

## Foreign Keys และ Referential Integrity

### Foreign Key คืออะไร?

**Foreign Key (FK)** คือ column ที่ชี้ไปหา Primary Key ของอีกตาราง มันทำหน้าที่:
1. สร้างลิงก์ระหว่างตาราง
2. บังคับ Referential Integrity

```sql
-- ตัวอย่าง Foreign Key constraints

-- employees.dept_id → departments.dept_id
ALTER TABLE employees 
ADD CONSTRAINT fk_emp_dept 
FOREIGN KEY (dept_id) REFERENCES departments(dept_id);

-- employees.manager_id → employees.emp_id (Self-referential)
ALTER TABLE employees 
ADD CONSTRAINT fk_emp_manager 
FOREIGN KEY (manager_id) REFERENCES employees(emp_id);

-- orders.customer_id → customers.customer_id
ALTER TABLE orders 
ADD CONSTRAINT fk_order_customer 
FOREIGN KEY (customer_id) REFERENCES customers(customer_id);
```

### Referential Integrity คืออะไร?

**Referential Integrity** คือกฎที่บังคับว่า Foreign Key ต้องชี้ไปหา Primary Key ที่มีอยู่จริง หรือเป็น NULL

```sql
-- ลองทดสอบ Referential Integrity

-- ERROR! dept_id = 99 ไม่มีในตาราง departments
INSERT INTO employees (emp_id, first_name, last_name, dept_id)
VALUES (999, 'Test', 'User', 99);
-- ERROR: insert or update on table "employees" violates foreign key constraint

-- OK: dept_id = 1 มีอยู่จริง
INSERT INTO employees (emp_id, first_name, last_name, dept_id)
VALUES (999, 'Test', 'User', 1);
-- INSERT 0 1 ✓

-- ERROR! ลบ department ที่มีพนักงานอยู่
DELETE FROM departments WHERE dept_id = 1;
-- ERROR: update or delete on table "departments" violates foreign key constraint

-- ต้องลบพนักงานก่อน ถึงจะลบ department ได้
DELETE FROM employees WHERE dept_id = 1;
DELETE FROM departments WHERE dept_id = 1;
```

### ON DELETE และ ON UPDATE Options

```sql
-- CASCADE: ลบ parent → ลบ children ด้วยอัตโนมัติ
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id)
        ON DELETE CASCADE  -- ลบ customer → ลบ orders ทั้งหมดด้วย
);

-- SET NULL: ลบ parent → set FK เป็น NULL
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    dept_id INT REFERENCES departments(dept_id)
        ON DELETE SET NULL  -- ลบ department → dept_id เป็น NULL
);

-- RESTRICT (default): ลบ parent ไม่ได้ถ้ายังมี children
-- NO ACTION: คล้าย RESTRICT แต่ตรวจสอบหลัง transaction
```

---

## Junction Tables (Bridge Tables)

### เมื่อ Many-to-Many ต้องการ Junction Table

```sql
-- ตัวอย่าง: พนักงานกับ projects (Many-to-Many)
CREATE TABLE projects (
    project_id INT PRIMARY KEY,
    project_name VARCHAR(100),
    start_date DATE,
    end_date DATE,
    budget DECIMAL(15,2)
);

-- Junction Table สำหรับ employees ↔ projects
CREATE TABLE employee_projects (
    emp_id INT REFERENCES employees(emp_id),
    project_id INT REFERENCES projects(project_id),
    role VARCHAR(50),           -- ข้อมูลเพิ่มเติม
    assigned_date DATE,         -- ข้อมูลเพิ่มเติม
    PRIMARY KEY (emp_id, project_id)  -- Composite Primary Key
);

-- Insert ข้อมูล: พนักงาน 1 คน ทำหลาย project
INSERT INTO employee_projects VALUES
(5, 1, 'Lead Developer', '2024-01-01'),
(5, 2, 'Contributor', '2024-02-01'),
(6, 1, 'Developer', '2024-01-15'),
(7, 3, 'Developer', '2024-03-01');
```

### Composite Primary Key ใน Junction Table

```sql
-- Composite PK ป้องกัน duplicate relationships
-- emp_id=5, project_id=1 ใส่ได้แค่ครั้งเดียว
INSERT INTO employee_projects VALUES (5, 1, 'Tester', '2024-04-01');
-- ERROR: duplicate key value violates unique constraint
```

---

## Cardinality Notation

**Cardinality** คือการระบุจำนวน "มากเท่าไหร่" ของแต่ละฝั่งใน relationship

### Crow's Foot Notation (สัญลักษณ์ที่ใช้บ่อย)

```
สัญลักษณ์ Crow's Foot:

│──    = One (exactly one)
│──<   = One or more (one-to-many)
○──    = Zero or one (optional one)
○──<  = Zero or more (optional many)

ตัวอย่าง:

departments              employees
┌──────────┐             ┌─────────┐
│          │──────────<  │         │
└──────────┘     1   N   └─────────┘
1 department has 0 or more employees
```

### Chen Notation

```
Chen Notation:
- Diamond = Relationship
- Rectangle = Entity
- Oval = Attribute
- Lines with numbers/letters = Cardinality

departments ---< 1:N >--- employees
customers ---< 1:N >--- orders ---< 1:N >--- order_items ---< N:1 >--- products
```

---

## ตัวอย่างการตรวจสอบความสัมพันธ์ด้วย SQL

```sql
-- 1. ดูโครงสร้าง Foreign Key ใน PostgreSQL
SELECT
    tc.constraint_name,
    tc.table_name AS from_table,
    kcu.column_name AS from_column,
    ccu.table_name AS to_table,
    ccu.column_name AS to_column
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
    ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
ORDER BY tc.table_name;

-- ผลลัพธ์จะแสดง:
-- employees.dept_id → departments.dept_id
-- employees.manager_id → employees.emp_id
-- orders.customer_id → customers.customer_id
-- order_items.order_id → orders.order_id
-- order_items.product_id → products.product_id

-- 2. นับจำนวนระเบียนในแต่ละตาราง
SELECT 'departments' AS table_name, COUNT(*) AS row_count FROM departments
UNION ALL
SELECT 'employees', COUNT(*) FROM employees
UNION ALL
SELECT 'customers', COUNT(*) FROM customers
UNION ALL
SELECT 'products', COUNT(*) FROM products
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
UNION ALL
SELECT 'order_items', COUNT(*) FROM order_items;

-- 3. ตรวจสอบ Orphan Records (ระเบียนที่ไม่มี parent)
-- employees ที่ไม่มี department
SELECT emp_id, first_name, last_name
FROM employees
WHERE dept_id IS NULL;

-- orders ที่ไม่มี customer (ถ้า FK ไม่ strict)
SELECT order_id 
FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM customers c WHERE c.customer_id = o.customer_id
);
```

---

## ความสัมพันธ์แบบ Self-Referential (Recursive)

ใน employees มี `manager_id` ที่ชี้กลับมาที่ `emp_id` ในตารางเดียวกัน:

```
Self-Referential Relationship:

employees
┌─────────────────┐
│ emp_id     PK   │◄──┐
│ first_name      │   │
│ last_name       │   │ manager_id ชี้กลับมา
│ manager_id FK   │───┘ (FK → PK ในตารางเดียวกัน)
│ ...             │
└─────────────────┘

โครงสร้าง Hierarchy:
CEO (emp_id=1, manager_id=NULL)
├── CTO (emp_id=2, manager_id=1)
│   ├── VP Engineering (emp_id=4, manager_id=2)
│   │   ├── Senior Engineer (emp_id=5, manager_id=4)
│   │   ├── Senior Engineer (emp_id=6, manager_id=4)
│   │   └── Engineer (emp_id=7, manager_id=4)
│   └── R&D Director (emp_id=37, manager_id=2)
└── CFO (emp_id=3, manager_id=1)
    └── Finance Director (emp_id=25, manager_id=1)
```

```sql
-- ดู hierarchy ของพนักงาน (ใช้ Self JOIN)
SELECT 
    e.emp_id,
    e.first_name || ' ' || e.last_name AS employee,
    e.job_title,
    m.first_name || ' ' || m.last_name AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
ORDER BY e.manager_id NULLS FIRST, e.emp_id;
```

---

## สรุปเปรียบเทียบประเภทความสัมพันธ์

```
┌──────────────┬──────────────────┬──────────────────────┬──────────────────────┐
│ ประเภท       │ ตัวอย่าง         │ FK อยู่ที่           │ วิธีระบุ             │
├──────────────┼──────────────────┼──────────────────────┼──────────────────────┤
│ One-to-One   │ emp ↔ profile    │ ฝั่งใดฝั่งหนึ่ง     │ FK + UNIQUE          │
│ One-to-Many  │ dept → employees │ Child table (Many)   │ FK ในตาราง Many      │
│ Many-to-Many │ orders ↔ products│ Junction Table       │ 2 FK ใน Junction     │
│ Self-referent│ emp → manager    │ Same table           │ FK ชี้ PK ตัวเอง    │
└──────────────┴──────────────────┴──────────────────────┴──────────────────────┘
```

---

## แนวทางการออกแบบ Relationship

### หลักการเลือก Relationship

```
ถามตัวเอง 2 คำถาม:

1. "A หนึ่งรายการ เชื่อมกับ B ได้กี่รายการ?"
   - ถ้าตอบ "แค่ 1" → A side เป็น "One"
   - ถ้าตอบ "หลายรายการ" → A side เป็น "Many"

2. "B หนึ่งรายการ เชื่อมกับ A ได้กี่รายการ?"
   - ถ้าตอบ "แค่ 1" → B side เป็น "One"
   - ถ้าตอบ "หลายรายการ" → B side เป็น "Many"

ตัวอย่าง employees ↔ departments:
Q1: "Department 1 อัน มีพนักงานกี่คน?" → หลายคน → Department = "One"
Q2: "Employee 1 คน อยู่กี่ Department?" → แค่ 1 → Employee = "Many"
→ departments (1) : employees (N)
```

---

## แบบฝึกหัดภาค 21

**ข้อ 1:** จากฐานข้อมูลตัวอย่าง จงระบุว่า orders กับ customers มีความสัมพันธ์แบบใด และ FK อยู่ที่ตารางใด

```sql
-- เฉลย: One-to-Many (customers 1 : orders N)
-- FK: orders.customer_id → customers.customer_id
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    COUNT(o.order_id) AS total_orders
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, customer_name
HAVING COUNT(o.order_id) > 0
ORDER BY total_orders DESC
LIMIT 5;
```

**ข้อ 2:** order_items เป็น Junction Table ระหว่างตารางใด? จงอธิบาย

```sql
-- เฉลย: order_items เป็น Junction Table ระหว่าง orders และ products
-- ทำให้เกิด Many-to-Many relationship
-- 1 order มีได้หลาย products
-- 1 product อยู่ได้ในหลาย orders
SELECT 
    o.order_id,
    COUNT(DISTINCT oi.product_id) AS products_in_order,
    p.product_id,
    COUNT(DISTINCT oi.order_id) AS orders_with_product
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY o.order_id, p.product_id
LIMIT 10;
```

**ข้อ 3:** manager_id ใน employees เป็น Self-Referential relationship อย่างไร? แสดงด้วย SQL

```sql
-- เฉลย: manager_id FK → emp_id PK ในตารางเดียวกัน
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee,
    e.job_title,
    CONCAT(m.first_name, ' ', m.last_name) AS reports_to,
    m.job_title AS manager_title
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
WHERE e.manager_id IS NOT NULL
ORDER BY m.emp_id, e.emp_id;
```

**ข้อ 4:** จงนับว่าแต่ละ Department มีพนักงานกี่คน รวมถึง Department ที่ไม่มีพนักงาน

```sql
-- เฉลย
SELECT 
    d.dept_id,
    d.dept_name,
    d.location,
    COUNT(e.emp_id) AS employee_count,
    COALESCE(SUM(e.salary), 0) AS total_salary_budget
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
GROUP BY d.dept_id, d.dept_name, d.location
ORDER BY employee_count DESC;
```

**ข้อ 5:** หา customer ที่สั่งสินค้ามากกว่า 2 ครั้ง พร้อมยอดรวม

```sql
-- เฉลย
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.city,
    COUNT(o.order_id) AS order_count,
    SUM(o.total_amount) AS total_spent,
    MIN(o.order_date) AS first_order,
    MAX(o.order_date) AS last_order
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, customer_name, c.city
HAVING COUNT(o.order_id) > 2
ORDER BY order_count DESC;
```

**ข้อ 6:** สร้าง Junction Table สำหรับ employees กับ skills (Many-to-Many)

```sql
-- เฉลย
CREATE TABLE skills (
    skill_id INT PRIMARY KEY,
    skill_name VARCHAR(100) NOT NULL,
    category VARCHAR(50)
);

CREATE TABLE employee_skills (
    emp_id INT REFERENCES employees(emp_id),
    skill_id INT REFERENCES skills(skill_id),
    proficiency_level VARCHAR(20) CHECK (proficiency_level IN ('beginner', 'intermediate', 'advanced', 'expert')),
    certified BOOLEAN DEFAULT FALSE,
    acquired_date DATE,
    PRIMARY KEY (emp_id, skill_id)
);

INSERT INTO skills VALUES
(1, 'Python', 'Programming'),
(2, 'SQL', 'Database'),
(3, 'Java', 'Programming'),
(4, 'Project Management', 'Management'),
(5, 'Data Analysis', 'Analytics');

INSERT INTO employee_skills VALUES
(5, 1, 'advanced', TRUE, '2018-01-01'),
(5, 2, 'expert', TRUE, '2016-06-01'),
(6, 1, 'intermediate', FALSE, '2019-03-01'),
(6, 3, 'advanced', TRUE, '2017-01-01');
```

**ข้อ 7:** ตรวจสอบว่ามี order_items ที่มี order_id ไม่มีใน orders หรือไม่

```sql
-- เฉลย: ตรวจหา Orphan Records
SELECT oi.item_id, oi.order_id
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_id IS NULL;
-- ถ้า FK ถูก enforce ก็จะไม่มีผลลัพธ์

-- อีกวิธี
SELECT oi.item_id, oi.order_id
FROM order_items oi
WHERE oi.order_id NOT IN (SELECT order_id FROM orders);
```

**ข้อ 8:** แสดงสายบังคับบัญชาของพนักงานระดับ "Engineer" ว่าใครเป็นผู้จัดการ

```sql
-- เฉลย
SELECT 
    e.emp_id,
    CONCAT(e.first_name, ' ', e.last_name) AS engineer_name,
    e.job_title,
    CONCAT(m.first_name, ' ', m.last_name) AS direct_manager,
    m.job_title AS manager_title,
    CONCAT(gm.first_name, ' ', gm.last_name) AS grand_manager,
    gm.job_title AS grand_manager_title
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
LEFT JOIN employees gm ON m.manager_id = gm.emp_id
WHERE e.job_title LIKE '%Engineer%'
ORDER BY e.emp_id;
```

**ข้อ 9:** หา products ทั้งหมดที่ถูกสั่งซื้อ พร้อมจำนวนครั้งที่ถูกสั่ง

```sql
-- เฉลย
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    COUNT(DISTINCT oi.order_id) AS times_ordered,
    SUM(oi.quantity) AS total_quantity_sold,
    SUM(oi.quantity * oi.unit_price - oi.discount) AS total_revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category, p.price
ORDER BY times_ordered DESC, total_revenue DESC;
```

**ข้อ 10:** ออกแบบ ERD สำหรับระบบ School ที่มี students, teachers, courses, enrollments

```sql
-- เฉลย: ออกแบบตาราง

CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    enrollment_year INT
);

CREATE TABLE teachers (
    teacher_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    department VARCHAR(50),
    email VARCHAR(100) UNIQUE
);

CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    course_name VARCHAR(100) NOT NULL,
    credits INT,
    teacher_id INT REFERENCES teachers(teacher_id)  -- 1:N (1 teacher, many courses)
);

-- Junction Table: students ↔ courses (M:N)
CREATE TABLE enrollments (
    student_id INT REFERENCES students(student_id),
    course_id INT REFERENCES courses(course_id),
    enrollment_date DATE,
    grade VARCHAR(2),
    PRIMARY KEY (student_id, course_id)
);

-- Relationships:
-- teachers 1:N courses (1 teacher สอนได้หลาย courses)
-- students M:N courses (นักเรียนลงทะเบียนได้หลาย courses)
-- (enrollments เป็น Junction Table)
```

---

## สรุปภาค 21

ในภาคนี้เราได้เรียนรู้:

1. **One-to-One (1:1)** — ความสัมพันธ์แบบหนึ่งต่อหนึ่ง ใช้ UNIQUE constraint บน FK
2. **One-to-Many (1:N)** — พบมากที่สุด FK อยู่ที่ Child Table
3. **Many-to-Many (M:N)** — ต้องใช้ Junction Table มี FK สองตัว
4. **Self-Referential** — FK ชี้กลับมาที่ PK ในตารางเดียวกัน
5. **Foreign Keys** — สร้างลิงก์ระหว่างตาราง บังคับ Referential Integrity
6. **ERD Diagrams** — วิธีวาดภาพความสัมพันธ์ของตาราง

**ในภาคต่อไป** เราจะเริ่มเรียน JOIN จริงๆ โดยเริ่มจาก INNER JOIN ซึ่งเป็น JOIN ที่ใช้บ่อยที่สุด!
