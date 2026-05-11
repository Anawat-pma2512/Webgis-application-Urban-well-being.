# Bangkok Urban Well-being WebGIS — System Design Document

## 1. วัตถุประสงค์ของระบบ (System Objectives)

พัฒนาระบบ WebGIS เพื่อวิเคราะห์และแสดงผลข้อมูลด้าน **Urban Well-being** ของกรุงเทพมหานคร ครอบคลุม 3 ประเด็นหลัก:

- **Transportation Access** — วิเคราะห์ระดับการเข้าถึงระบบขนส่งมวลชนทางราง (BTS/MRT) ในแต่ละเขต
- **Food Access** — ประเมินศักยภาพพื้นที่เกษตรกรรมและแหล่งผลิตอาหารในเชิงพื้นที่
- **Disaster Risk** — ระบุพื้นที่เสี่ยงภัยพิบัติ (น้ำท่วม พื้นที่ลุ่มต่ำ) จากข้อมูลการใช้ประโยชน์ที่ดิน

---

## 2. โครงสร้างระบบ (System Architecture)

```
┌────────────────────────────────────────────────────────┐
│                    CLIENT (Browser)                    │
│  ┌───────────┐ ┌───────────┐ ┌────────────────────┐    │
│  │  Leaflet  │ │ UI/CSS    │ │  Analysis Engine   │    │
│  │  Map Core │ │ Dashboard │ │  (JavaScript)      │    │
│  └─────┬─────┘ └─────┬─────┘ └────────┬───────────┘    │
│        │             │                │                │
│        └─────────────┴────────────────┘                │
│                      │                                 │
│              ┌───────┴───────┐                         │
│              │  GeoJSON Data │                         │
│              │  (Static)     │                         │
│              └───────────────┘                         │
└────────────────────────────────────────────────────────┘

Data Files:
├── index.html              (Main application)
├── transit_stations.geojson (BTS+MRT 139 stations)
├── mrt_lines.geojson       (MRT rail lines)
├── districts.geojson       (50 district boundaries)
├── bkk_boundary.geojson    (Bangkok province boundary)
├── food_areas.geojson      (Agricultural land dissolved)
├── risk_areas.geojson      (Water/flood-prone areas)
└── district_stats.json     (Pre-computed statistics)
```

**Technology Stack:**
- **Frontend**: HTML5 + CSS3 + Vanilla JavaScript
- **Map Library**: Leaflet.js 1.9.4
- **Basemap**: CartoDB Dark (dark_all)
- **Fonts**: Google Fonts — Sarabun + Prompt (Thai-optimized)
- **Hosting**: GitHub Pages (static deployment)

---

## 3. Workflow

```
[Raw Data]                  [Processing]              [Visualization]
                
Shapefiles ──┐                                       ┌── Interactive Map
  - LULC     │              Python/GeoPandas          │   - Layer toggle
  - Districts├──→ CRS Transform (UTM47→WGS84) ──→ GeoJSON ──→ Popup info
  - Boundary │   Simplification                  │   - Filter/Zoom
             │   Category Classification         │   
CSV Files ───┤   Spatial Join (Station↔District) ├── Dashboard
  - BTS      │   Statistical Aggregation         │   - Stat cards
  - MRT      │                                   │   - Bar charts
             │                                   │
Shapefiles ──┘                                   └── Analysis Report
  - Roads                                            - Patterns
  - MRT Lines                                        - Insights
                                                     - Recommendations
```

### Processing Steps:
1. **Data Ingestion** — อ่าน Shapefiles (SHP/DBF/PRJ) และ CSV
2. **CRS Transformation** — แปลง EPSG:32647 (UTM Zone 47N) → EPSG:4326 (WGS84)
3. **Geometry Simplification** — ลดความซับซ้อนของ polygon เพื่อประสิทธิภาพ web
4. **LULC Classification** — จัดหมวดหมู่การใช้ที่ดิน 98 ประเภท → 10 กลุ่มหลัก
5. **Spatial Join** — เชื่อมสถานีรถไฟฟ้ากับเขตปกครอง
6. **Statistical Aggregation** — คำนวณสัดส่วนพื้นที่ต่อเขต
7. **GeoJSON Export** — ส่งออกเป็น GeoJSON สำหรับ Leaflet
8. **Web Rendering** — แสดงผลเชิงโต้ตอบบน Browser

---

## 4. ประเภทข้อมูล (Data Types)

| ชั้นข้อมูล | ไฟล์ต้นทาง | Geometry | CRS เดิม | จำนวน Features | ขนาด |
|---|---|---|---|---|---|
| สถานี BTS | bts_station.csv | Point | WGS84 | 63 | 28 KB |
| สถานี MRT | mrt_station.csv | Point | WGS84 | 76 | 28 KB |
| เส้นทาง MRT | mrt_line.shp | LineString | UTM47 | 3 | 8 KB |
| ถนน | THA_roads.shp | LineString/Multi | WGS84 | 4,107 | 788 KB |
| LULC กรุงเทพ | LU_BKK_2566.shp | Polygon | UTM47 | 11,179 | 5.7 MB |
| ขอบเขตเขต | Amp.shp | Polygon | UTM47 | 53 | 686 KB |
| ขอบเขตแขวง | Tam.shp | Polygon | UTM47 | 182 | 1.3 MB |
| ขอบเขตจังหวัด | Pro.shp | Polygon | UTM47 | 1 | 30 KB |

### LULC Classification Mapping:

| รหัส | หมวดหมู่ | คำอธิบาย | จำนวน |
|---|---|---|---|
| A0-A9 | agriculture | เกษตรกรรมทุกประเภท | ~4,648 |
| F3 | forest | ป่าชายเลน | ~2 |
| M1-M2 | open_green | ทุ่งหญ้า/พื้นที่ชุ่มน้ำ | ~1,423 |
| M3-M7 | waste_hazard | หลุมขุด/ที่ทิ้งขยะ | ~343 |
| U1-U2 | residential_commercial | เมือง/ชุมชน | ~2,654 |
| U3 | institutional | สถาบัน/ราชการ | ~686 |
| U4 | transportation | ถนน/ทางรถไฟ/สนามบิน | ~518+ |
| U5 | industrial | โรงงาน/นิคม | ~539 |
| U6-U7 | recreation | สนามกอล์ฟ/สวน | ~343+ |
| W1-W2 | water | แม่น้ำ/คลอง/บึง | ~541 |

---

## 5. ฟีเจอร์ระบบ (System Features)

### 5.1 Interactive Map
- เปิด/ปิดชั้นข้อมูล 3 ชั้น (Transportation, Food, Disaster Risk)
- Popup แสดงรายละเอียดเมื่อคลิกจุด/พื้นที่
- Filter dropdown เลือกเขตปกครอง
- Zoom to location เมื่อเลือกเขต
- Mouse hover highlight บนเขต

### 5.2 Dashboard Panel
- สถิติภาพรวม (จำนวนสถานี, เขตที่มีเกษตร, เขตเสี่ยงสูง)
- Bar chart แสดงระดับ 4 ตัวชี้วัดเมื่อเลือกเขต
- ข้อมูลพื้นที่ (ไร่) ของแต่ละหมวด

### 5.3 Analysis & Interpretation
- สรุป Pattern จาก 3 ประเด็น
- ความเชื่อมโยงเชิงพื้นที่ (Spatial Nexus)
- ข้อเสนอแนวทางวางผังเมือง 5 ประเด็น


