#!/usr/bin/env python3
"""
generate_healthcare_with_dimensions.py
Generates:
- fact_healthcare.csv  (500,000 rows, 15 dimension IDs)
- 15 dimension CSV files (dim_patient.csv, dim_doctor.csv, etc.)
"""

import csv
import random
import datetime
import os
from faker import Faker

fake = Faker(['en_IN', 'en_US'])
random.seed(42)

# Row count
FACT_ROWS = 500_000

# % of repeat patients
REPEAT_PATIENT_PCT = 0.10

# Output folder
OUT_DIR = "healthcare_dataset"
os.makedirs(OUT_DIR, exist_ok=True)

# Dimension sizes
DIM_SIZES = {
    "patient": 450_000,   # many patients but some repeats
    "doctor": 500,
    "equipment": 300,
    "department": 50,
    "diagnosis": 200,
    "insurance": 50,
    "procedure": 300,
    "medical": 100,
    "bed": 200,
    "staff": 800,
    "location": 150,
    "visitor": 1000,
    "room": 400,
    "pharmacy": 120,
    "lab": 150
}

# Store dimension data
dim_data = {}

# Generate dimensions
def gen_dimensions():
    dim_data["patient"] = []
    for pid in range(1, DIM_SIZES["patient"] + 1):
        dob = fake.date_of_birth(minimum_age=0, maximum_age=90)
        age = (datetime.date.today() - dob).days // 365
        dim_data["patient"].append([
            pid, fake.name(), random.choice(['M', 'F', 'Other']),
            dob.isoformat(), age,
            random.choice(['A+','A-','B+','B-','O+','O-','AB+','AB-']),
            random.choice(['Single','Married','Divorced','Widowed'])
        ])

    dim_data["doctor"] = [[did, fake.name(), random.choice(
        ['Cardiology','Neurology','Orthopedics','Pediatrics','Oncology','General Medicine','Emergency']
    )] for did in range(1, DIM_SIZES["doctor"]+1)]

    dim_data["equipment"] = [[eid, f"Equip-{eid:03d}", random.choice(
        ['MRI','X-Ray','Ventilator','ECG','Ultrasound']
    )] for eid in range(1, DIM_SIZES["equipment"]+1)]

    dim_data["department"] = [[did, f"Dept-{did:02d}", random.choice(
        ['Ward A','Ward B','Ward C','ICU','OPD']
    )] for did in range(1, DIM_SIZES["department"]+1)]

    dim_data["diagnosis"] = [[did, f"DX-{did:03d}", random.choice(
        ['Hypertension','Diabetes','Asthma','UTI','Back Pain','Flu','Covid-19']
    )] for did in range(1, DIM_SIZES["diagnosis"]+1)]

    dim_data["insurance"] = [[iid, random.choice(
        ['Aetna','BlueShield','Religare','Max Bupa','LIC','None']
    ), fake.company()] for iid in range(1, DIM_SIZES["insurance"]+1)]

    dim_data["procedure"] = [[pid, f"PROC-{pid:04d}", f"Procedure {pid}"] for pid in range(1, DIM_SIZES["procedure"]+1)]

    dim_data["medical"] = [[mid, f"Med-{mid}", random.choice(
        ['Tablet','Injection','Syrup','Capsule']
    )] for mid in range(1, DIM_SIZES["medical"]+1)]

    dim_data["bed"] = [[bid, random.choice(
        ['General','Semi-Private','Private','ICU']
    ), random.randint(1,6)] for bid in range(1, DIM_SIZES["bed"]+1)]

    dim_data["staff"] = [[sid, fake.name(), random.choice(
        ['Nurse','Technician','Receptionist','Cleaner']
    )] for sid in range(1, DIM_SIZES["staff"]+1)]

    dim_data["location"] = [[lid, fake.city(), fake.state()] for lid in range(1, DIM_SIZES["location"]+1)]

    dim_data["visitor"] = [[vid, fake.name(), fake.phone_number()] for vid in range(1, DIM_SIZES["visitor"]+1)]

    dim_data["room"] = [[rid, random.randint(100,999), random.choice(['A','B','C','D'])] for rid in range(1, DIM_SIZES["room"]+1)]

    dim_data["pharmacy"] = [[pid, fake.company(), fake.street_name()] for pid in range(1, DIM_SIZES["pharmacy"]+1)]

    dim_data["lab"] = [[lid, fake.company(), random.choice(['Blood Test','MRI','X-Ray','CT Scan'])] for lid in range(1, DIM_SIZES["lab"]+1)]

# Write dimension files
def write_dimensions():
    headers = {
        "patient": ["patient_id","patient_name","gender","dob","age","blood_group","marital_status"],
        "doctor": ["doctor_id","doctor_name","specialization"],
        "equipment": ["equipment_id","equipment_name","equipment_type"],
        "department": ["department_id","department_name","ward"],
        "diagnosis": ["diagnosis_id","diagnosis_code","diagnosis_desc"],
        "insurance": ["insurance_id","insurance_name","insurance_company"],
        "procedure": ["procedure_id","procedure_code","procedure_desc"],
        "medical": ["medical_id","medical_name","medical_type"],
        "bed": ["bed_id","bed_type","bed_number"],
        "staff": ["staff_id","staff_name","staff_role"],
        "location": ["location_id","city","state"],
        "visitor": ["visitor_id","visitor_name","visitor_contact"],
        "room": ["room_id","room_number","block"],
        "pharmacy": ["pharmacy_id","pharmacy_name","street"],
        "lab": ["lab_id","lab_name","lab_service"]
    }
    for dim, rows in dim_data.items():
        with open(os.path.join(OUT_DIR, f"dim_{dim}.csv"), "w", newline='', encoding='utf-8') as f:
            writer = csv.writer(f)
            writer.writerow(headers[dim])
            writer.writerows(rows)

# Generate fact table
def gen_fact():
    fact_file = os.path.join(OUT_DIR, "fact_healthcare.csv")
    headers = [
        "patient_id","doctor_id","equipment_id","department_id","diagnosis_id","insurance_id",
        "procedure_id","medical_id","bed_id","staff_id","location_id","visitor_id",
        "room_id","pharmacy_id","lab_id","admission_date","discharge_date","length_of_stay",
        "billed_amount","paid_amount","claim_amount","payment_method"
    ]
    with open(fact_file, "w", newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerow(headers)

        # Preload patient IDs for repeat logic
        patient_ids = [p[0] for p in dim_data["patient"]]
        repeat_count = int(FACT_ROWS * REPEAT_PATIENT_PCT)
        repeat_patients = random.sample(patient_ids, repeat_count)

        for i in range(FACT_ROWS):
            # Assign patient (repeat or new)
            if random.random() < REPEAT_PATIENT_PCT:
                pid = random.choice(repeat_patients)
            else:
                pid = random.choice(patient_ids)

            row = [
                pid,
                random.randint(1, DIM_SIZES["doctor"]),
                random.randint(1, DIM_SIZES["equipment"]),
                random.randint(1, DIM_SIZES["department"]),
                random.randint(1, DIM_SIZES["diagnosis"]),
                random.randint(1, DIM_SIZES["insurance"]),
                random.randint(1, DIM_SIZES["procedure"]),
                random.randint(1, DIM_SIZES["medical"]),
                random.randint(1, DIM_SIZES["bed"]),
                random.randint(1, DIM_SIZES["staff"]),
                random.randint(1, DIM_SIZES["location"]),
                random.randint(1, DIM_SIZES["visitor"]),
                random.randint(1, DIM_SIZES["room"]),
                random.randint(1, DIM_SIZES["pharmacy"]),
                random.randint(1, DIM_SIZES["lab"]),
            ]
            # Dates & measures
            admission = fake.date_between(start_date="-5y", end_date="today")
            discharge = admission + datetime.timedelta(days=random.randint(0, 14))
            length_of_stay = (discharge - admission).days
            billed_amount = round(random.uniform(1000, 50000), 2)
            claim_amount = round(random.uniform(0, billed_amount), 2)
            paid_amount = round(random.uniform(0, billed_amount), 2)
            payment_method = random.choice(['Cash','Card','UPI','Insurance','Other'])

            row.extend([
                admission.isoformat(),
                discharge.isoformat(),
                length_of_stay,
                billed_amount,
                paid_amount,
                claim_amount,
                payment_method
            ])
            writer.writerow(row)

if __name__ == "__main__":
    print("Generating dimensions...")
    gen_dimensions()
    write_dimensions()
    print("Generating fact table...")
    gen_fact()
    print(f"Done! Files saved in folder: {OUT_DIR}")
