#!/usr/bin/env python3
"""
generate_hr_70cols.py
Generates a large synthetic HR CSV with 70 meaningful columns, manager relationships,
logical dates, and ~10% repeated employees (same demographics, new snapshots).

Usage:
  python generate_hr_70cols.py --rows 1000000 --outfile hr_1M.csv --chunk 10000 --tqdm

Notes:
- Use --outfile filename ending with .gz to write gzipped CSV.
- Adjust --repeat_prob for percent of repeated employees (default 0.10).
"""
import csv
import random
import argparse
import datetime
import gzip
from faker import Faker
import numpy as np

fake = Faker(['en_IN', 'en_US'])

# default probability an output row re-uses an existing employee (same emp_id, name, dob, etc)
REPEAT_PROB_DEFAULT = 0.10
# initial leadership count so managers exist from the start
INITIAL_LEADERS = 50

random.seed(42)
np.random.seed(42)

# designation hierarchy: index 0 = highest rank
DESIGNATION_HIERARCHY = [
    "CEO", "VP", "Director", "Senior Director", "Associate Director",
    "Head", "Sr Manager", "Manager", "Senior Engineer", "Lead", "Senior",
    "Engineer", "Analyst", "Associate", "Intern"
]
# map designation -> rank index for quick comparison (lower is higher)
DESIG_TO_RANK = {d: i for i, d in enumerate(DESIGNATION_HIERARCHY)}

def random_date(start_year=1990, end_year=2025):
    start = datetime.date(start_year, 1, 1)
    end = datetime.date(end_year, 12, 31)
    delta = end - start
    return start + datetime.timedelta(days=random.randint(0, delta.days))

class EmployeePool:
    """
    Stores created employees (stable demographics). Allows picking a manager
    who has a higher designation rank.
    """
    def __init__(self):
        self.employees = []  # list of dicts representing stable employee info
        self.next_seq = 1

    def create_new_employee(self, force_designation=None, min_rank=None):
        """
        Create a new employee with stable demographic fields.
        If force_designation given, set that designation.
        If min_rank is given, choose designation with rank <= min_rank (higher-level).
        """
        seq = self.next_seq
        self.next_seq += 1
        emp_id = f"E{seq:07d}"
        full_name = fake.name()
        gender = random.choice(['M', 'F', 'Other'])
        dob = random_date(1955, 2002)
        age = (datetime.date.today() - dob).days // 365
        # stable contact/demographic fields
        city = fake.city()
        state = fake.state()
        pincode = fake.postcode()
        email = fake.email()
        personal_email = fake.free_email()
        phone = fake.phone_number()
        emergency_contact = fake.name()
        emergency_relation = random.choice(['Spouse','Parent','Sibling','Friend'])
        emergency_phone = fake.phone_number()
        marital_status = random.choice(['Single','Married','Divorced','Widowed'])
        nationality = random.choice(['Indian','American','British','Canadian','Australian'])
        highest_qualification = random.choice(['High School','Diploma','Bachelors','Masters','PhD'])
        university = fake.company() + " University"
        recruitment_source = random.choice(['LinkedIn','Referral','Job Portal','Campus','Agency','Direct'])
        joining_date = random_date(2008, 2025)
        # choose designation respecting min_rank or forced designation
        if force_designation:
            designation = force_designation
        else:
            if min_rank is not None:
                # choose designation with rank <= min_rank (i.e., equal or higher)
                choices = [d for d, r in DESIG_TO_RANK.items() if r <= min_rank]
                designation = random.choice(choices) if choices else random.choice(list(DESIG_TO_RANK.keys()))
            else:
                designation = random.choice(DESIGNATION_HIERARCHY[5:])  # default: mid-to-lower levels
        # stable department - keep for simplicity (can change in snapshots)
        department = random.choice(['HR','Finance','IT','Sales','Marketing','Operations','R&D','Customer Success','Legal'])
        # stable bank info
        bank_account = f"{random.randint(10000000,99999999)}"
        bank_ifsc = "IFSC" + str(random.randint(1000,9999))
        pf_number = f"PF{random.randint(1000000,9999999)}"
        gratuity_eligible = random.choice(['Yes','No'])
        is_leader = DESIG_TO_RANK.get(designation, 100) <= 3  # top 4 ranks considered leaders

        emp = {
            "emp_id": emp_id,
            "full_name": full_name,
            "gender": gender,
            "dob": dob,
            "age": age,
            "city": city,
            "state": state,
            "pincode": pincode,
            "email": email,
            "personal_email": personal_email,
            "phone": phone,
            "emergency_contact": emergency_contact,
            "emergency_relation": emergency_relation,
            "emergency_phone": emergency_phone,
            "marital_status": marital_status,
            "nationality": nationality,
            "highest_qualification": highest_qualification,
            "university": university,
            "recruitment_source": recruitment_source,
            "joining_date": joining_date,
            "designation": designation,
            "department": department,
            "bank_account": bank_account,
            "bank_ifsc": bank_ifsc,
            "pf_number": pf_number,
            "gratuity_eligible": gratuity_eligible,
            "is_leader": is_leader
        }
        self.employees.append(emp)
        return emp

    def pick_existing_employee(self):
        return random.choice(self.employees) if self.employees else None

    def pick_manager_for(self, emp_designation):
        """
        Pick a manager who has a strictly higher rank (i.e., rank index < emp's rank index).
        If multiple available, choose randomly.
        If none available, create a leader on the fly (rare).
        """
        emp_rank = DESIG_TO_RANK.get(emp_designation, len(DESIGNATION_HIERARCHY)-1)
        candidates = [e for e in self.employees if DESIG_TO_RANK.get(e['designation'], 100) < emp_rank]
        if candidates:
            return random.choice(candidates)
        # fallback: create a leader with rank better than emp_rank (if possible)
        better_rank = max(0, emp_rank - 3)
        # force a creation with a designation that has rank <= better_rank
        possible_designations = [d for d,r in DESIG_TO_RANK.items() if r < emp_rank]
        force_designation = random.choice(possible_designations) if possible_designations else "Director"
        return self.create_new_employee(force_designation=force_designation, min_rank=None)

    def population(self):
        return len(self.employees)

POOL = EmployeePool()

# HEADER with 70 columns (counted carefully)
HEADER = [
    "emp_id", "employee_code", "full_name", "gender", "dob", "age", "personal_email", "work_email",
    "phone", "emergency_contact", "emergency_relation", "emergency_phone", "marital_status",
    "nationality", "city", "state", "pincode", "address", "highest_qualification", "university",
    "joining_date", "hire_date", "employment_type", "department", "location", "work_mode",
    "designation", "designation_rank", "manager_id", "manager_name", "manager_designation",
    "manager_tenure_years", "salary_annual", "salary_currency", "salary_grade", "bonus_last_year",
    "stock_options", "last_promotion_date", "promotion_count", "promotion_eligible", "last_appraisal_score",
    "last_appraisal_rating", "review_submission_date", "review_approval_date", "reviewer_id", "reviewer_name",
    "total_experience_years", "years_in_company", "probation_end_date", "contract_end_date",
    "notice_period_days", "notice_given_date", "last_working_date", "termination_reason", "rehire_eligible",
    "pf_number", "provident_fund_contribution", "gratuity_eligible", "medical_insurance_provider",
    "bank_account", "bank_ifsc", "tax_id", "skills", "certifications", "trainings_attended",
    "leaves_balance", "leaves_taken_year", "sick_leaves_taken", "casual_leaves_taken", "maternity_leaves_taken",
    "paternity_leaves_taken", "warnings_count", "awards", "projects_count", "current_project",
    "onboarding_complete", "background_check_status", "record_created_at", "record_updated_at"
]
# HEADER length check (should be 70)
assert len(HEADER) == 70, f"HEADER must contain 70 columns, currently has {len(HEADER)}"

def months_between(d1, d2):
    # approximate months difference
    return max(0, (d2.year - d1.year) * 12 + (d2.month - d1.month))

def generate_snapshot_for_employee(emp, snapshot_index):
    """
    Given stable employee demographics (emp dict), generate a snapshot row with
    visit-like/temporal fields. snapshot_index used to create unique employee_code.
    """
    emp_id = emp["emp_id"]
    employee_code = f"EMP{snapshot_index+1:08d}"
    full_name = emp["full_name"]
    gender = emp["gender"]
    dob = emp["dob"]
    age = emp["age"]
    personal_email = emp["personal_email"]
    work_email = f"{full_name.lower().replace(' ','_')}.{emp_id.lower()}@examplecorp.com"
    phone = emp["phone"]
    emergency_contact = emp["emergency_contact"]
    emergency_relation = emp["emergency_relation"]
    emergency_phone = emp["emergency_phone"]
    marital_status = emp["marital_status"]
    nationality = emp["nationality"]
    city = emp["city"]
    state = emp["state"]
    pincode = emp["pincode"]
    address = fake.address().replace("\n", ", ")
    highest_qualification = emp["highest_qualification"]
    university = emp["university"]

    # hire date cannot be before dob + 18 years
    earliest_hire = dob + datetime.timedelta(days=365*18)
    hire_date = emp.get("joining_date", random_date(2008, 2025))
    # ensure hire_date is not earlier than earliest_hire
    if hire_date < earliest_hire:
        hire_date = earliest_hire + datetime.timedelta(days=random.randint(0, 365*2))

    employment_type = random.choice(['Full-Time','Part-Time','Contract','Intern'])
    department = emp.get("department", random.choice(['HR','Finance','IT','Sales','Marketing','Operations','R&D','Customer Success','Legal']))
    location = random.choice(['Head Office','Branch Office','Remote','Client Site'])
    work_mode = random.choice(['Onsite','Hybrid','Remote'])
    # designation: allow chance of promotion from stable emp designation or random
    if random.random() < 0.05:
        # small chance to pick slightly higher rank designation
        current_rank = DESIG_TO_RANK.get(emp.get("designation","Associate"), len(DESIGNATION_HIERARCHY)-1)
        new_rank = max(0, current_rank - random.randint(0,2))
        designation = DESIGNATION_HIERARCHY[new_rank]
    else:
        designation = emp.get("designation", random.choice(DESIGNATION_HIERARCHY[6:]))
    designation_rank = DESIG_TO_RANK.get(designation, len(DESIGNATION_HIERARCHY)-1)

    # Manager selection: pick from pool an employee with higher rank (strictly)
    manager = POOL.pick_manager_for(designation)
    manager_id = manager["emp_id"]
    manager_name = manager["full_name"]
    manager_designation = manager["designation"]
    manager_tenure_years = months_between(manager.get("joining_date", hire_date), datetime.date.today()) // 12

    # salary and comp
    base_salary = round(random.uniform(250000, 3500000), 2)
    salary_currency = random.choice(['INR','USD','GBP','EUR'])
    salary_grade = random.choice(['L1','L2','L3','M1','M2','S1','S2'])
    bonus_last_year = round(random.uniform(0, base_salary * 0.25), 2)
    stock_options = random.choice([0, 0, 0, random.randint(0,1000)])  # mostly 0
    # promotion/date logic
    promotion_count = random.randint(0, 5)
    last_promotion_date = ''
    if promotion_count > 0:
        # last promotion date must be after hire_date and before today
        lp = hire_date + datetime.timedelta(days=random.randint(180, max(365, (datetime.date.today()-hire_date).days)))
        if lp > datetime.date.today():
            lp = datetime.date.today() - datetime.timedelta(days=random.randint(0,365))
        last_promotion_date = lp

    promotion_eligible = random.choice(['Yes','No'])

    # appraisal & review dates
    last_appraisal_score = round(random.uniform(1.0, 5.0), 2)
    # map score to rating
    if last_appraisal_score >= 4.5:
        last_appraisal_rating = "Outstanding"
    elif last_appraisal_score >= 3.5:
        last_appraisal_rating = "Exceeds Expectations"
    elif last_appraisal_score >= 2.5:
        last_appraisal_rating = "Meets Expectations"
    else:
        last_appraisal_rating = "Needs Improvement"

    # review submission and approval: must be between hire_date and last_working_date (if any) or today
    window_end = datetime.date.today()
    review_submission_date = hire_date + datetime.timedelta(days=random.randint(30, max(30, (window_end - hire_date).days)))
    # approval at or after submission
    review_approval_date = review_submission_date + datetime.timedelta(days=random.randint(0, 30))
    reviewer = manager  # reviewer is manager for simplicity
    reviewer_id = reviewer['emp_id']
    reviewer_name = reviewer['full_name']

    # experience & company tenure calculations
    total_experience_years = round(random.uniform(0.0, 25.0), 1)
    years_in_company = round(months_between(hire_date, datetime.date.today()) / 12.0, 1)

    # probation and contract
    probation_end_date = hire_date + datetime.timedelta(days=90)
    contract_end_date = ''
    if employment_type == 'Contract':
        contract_end_date = hire_date + datetime.timedelta(days=random.randint(90, 365*3))

    notice_period_days = random.choice([0,30,60,90])
    notice_given_date = ''
    last_working_date = ''
    termination_reason = ''
    rehire_eligible = ''
    if random.random() < 0.08:
        # ~8% terminated/resigned
        last_working_date_date = hire_date + datetime.timedelta(days=random.randint(30, max(30, (datetime.date.today() - hire_date).days)))
        if last_working_date_date > datetime.date.today():
            last_working_date_date = datetime.date.today() - datetime.timedelta(days=random.randint(0,30))
        last_working_date = last_working_date_date
        termination_reason = random.choice(['Resignation','Retirement','Termination','Mutual Separation','Contract End'])
        rehire_eligible = random.choice(['Yes','No'])
        if random.random() < 0.5:
            notice_given_date = last_working_date - datetime.timedelta(days=random.choice([0,15,30,60]))
    else:
        last_working_date = ''
        termination_reason = ''
        rehire_eligible = 'Yes'  # still eligible by default

    # pf / gratuity
    pf_number = emp.get('pf_number', f"PF{random.randint(1000000,9999999)}")
    provident_fund_contribution = round(base_salary * random.uniform(0.01, 0.12), 2)
    gratuity_eligible = emp.get('gratuity_eligible', random.choice(['Yes','No']))
    medical_insurance_provider = random.choice(['Aetna','BlueCross','Religare','Max Bupa','None'])
    bank_account = emp.get('bank_account', f"{random.randint(10000000,99999999)}")
    bank_ifsc = emp.get('bank_ifsc', 'IFSC' + str(random.randint(1000,9999)))
    tax_id = f"TAX{random.randint(1000000,9999999)}"

    skills = ";".join(random.sample(['Excel','Power BI','Python','SQL','Communication','Leadership','Negotiation','Sales','Marketing','Cloud'], k=random.randint(1,4)))
    certifications = ";".join(random.sample(['PMP','AWS','Azure','Six Sigma','CFA','None'], k=1))
    trainings_attended = random.randint(0, 20)

    leaves_balance = random.randint(0, 40)
    leaves_taken_year = random.randint(0, 30)
    sick_leaves_taken = random.randint(0, 12)
    casual_leaves_taken = random.randint(0, 12)
    maternity_leaves_taken = random.randint(0, 1) if gender == 'F' else 0
    paternity_leaves_taken = random.randint(0, 1) if gender == 'M' else 0

    warnings_count = random.randint(0, 5)
    awards = random.choice(['Employee of the Month','Best Innovator','Top Performer','None'])
    projects_count = random.randint(0, 12)
    current_project = fake.bs()[:40]
    onboarding_complete = random.choice(['Yes','No'])
    background_check_status = random.choice(['Clear','Pending','Failed'])

    record_created_at = datetime.datetime.now().isoformat()
    record_updated_at = record_created_at

    # convert dates to isoformat strings or empty
    def d2s(d):
        if not d:
            return ''
        if isinstance(d, datetime.date):
            return d.isoformat()
        return str(d)

    row = [
        emp_id, employee_code, full_name, gender, d2s(dob), age, personal_email, work_email,
        phone, emergency_contact, emergency_relation, emergency_phone, marital_status,
        nationality, city, state, pincode, address, highest_qualification, university,
        d2s(emp.get('joining_date', hire_date)), d2s(hire_date), employment_type, department, location, work_mode,
        designation, designation_rank, manager_id, manager_name, manager_designation,
        manager_tenure_years, base_salary, salary_currency, salary_grade, bonus_last_year,
        stock_options, d2s(last_promotion_date), promotion_count, promotion_eligible, last_appraisal_score,
        last_appraisal_rating, d2s(review_submission_date), d2s(review_approval_date), reviewer_id, reviewer_name,
        total_experience_years, years_in_company, d2s(probation_end_date), d2s(contract_end_date),
        notice_period_days, d2s(notice_given_date), d2s(last_working_date), termination_reason, rehire_eligible,
        pf_number, provident_fund_contribution, gratuity_eligible, medical_insurance_provider,
        bank_account, bank_ifsc, tax_id, skills, certifications, trainings_attended,
        leaves_balance, leaves_taken_year, sick_leaves_taken, casual_leaves_taken, maternity_leaves_taken,
        paternity_leaves_taken, warnings_count, awards, projects_count, current_project,
        onboarding_complete, background_check_status, record_created_at, record_updated_at
    ]
    # final check: row must match HEADER length
    assert len(row) == len(HEADER), f"Row length {len(row)} != HEADER length {len(HEADER)}"
    return row

def main():
    parser = argparse.ArgumentParser(description="Generate synthetic HR CSV with 70 columns.")
    parser.add_argument('--rows', type=int, default=1000000, help='Number of rows to generate')
    parser.add_argument('--outfile', type=str, default='hr_1M.csv', help='Output CSV filename (use .gz to compress)')
    parser.add_argument('--chunk', type=int, default=10000, help='Rows per write chunk (reduce memory spikes)')
    parser.add_argument('--seed', type=int, default=42, help='Random seed')
    parser.add_argument('--tqdm', action='store_true', help='Show progress bar (requires tqdm)')
    parser.add_argument('--repeat_prob', type=float, default=REPEAT_PROB_DEFAULT, help='Probability a row uses an existing employee (0-1)')
    args = parser.parse_args()

    random.seed(args.seed)
    np.random.seed(args.seed)

    # create initial senior leaders so managers exist early
    for _ in range(INITIAL_LEADERS):
        # choose a high-level designation for leaders (CEO/VP/Director/...),
        # pick randomly among top 6 ranks to ensure availability of managers
        leader_designation = random.choice(DESIGNATION_HIERARCHY[:6])
        POOL.create_new_employee(force_designation=leader_designation)

    use_gzip = args.outfile.endswith('.gz')
    open_func = gzip.open if use_gzip else open
    mode = 'wt'

    with open_func(args.outfile, mode, newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerow(HEADER)

        try:
            if args.tqdm:
                from tqdm import tqdm
                pbar = tqdm(total=args.rows, desc='Generating rows')
            else:
                pbar = None
        except Exception:
            pbar = None

        pool = POOL

        for start in range(0, args.rows, args.chunk):
            batch = []
            end = min(start + args.chunk, args.rows)
            for i in range(start, end):
                # Decide whether to reuse an existing employee (repeat snapshot)
                do_repeat = (random.random() < args.repeat_prob) and (pool.population() > 0)
                if do_repeat:
                    emp = pool.pick_existing_employee()
                else:
                    # create a new employee; small chance to create a mid/high-level person
                    if random.random() < 0.02:
                        # create a manager-level employee
                        force_designation = random.choice(DESIGNATION_HIERARCHY[:8])
                        emp = pool.create_new_employee(force_designation=force_designation)
                    else:
                        emp = pool.create_new_employee()
                row = generate_snapshot_for_employee(emp, i)
                batch.append(row)
            writer.writerows(batch)
            if pbar:
                pbar.update(len(batch))

        if pbar:
            pbar.close()

    print(f"Done. Wrote {args.rows} rows to {args.outfile}")
    print(f"Unique employees created: {POOL.population()} (approx {100*(1-args.repeat_prob):.1f}% new expected)")

if __name__ == "__main__":
    main()
