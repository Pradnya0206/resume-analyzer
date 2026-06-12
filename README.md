# professional_smart_resume_analyzer_v2.py

import streamlit as st
import PyPDF2
import docx
import re
import matplotlib.pyplot as plt
from fpdf import FPDF

# =========================
# PAGE CONFIG
# =========================
st.set_page_config(page_title="AI Resume Analyzer", layout="wide")

# =========================
# TEXT EXTRACTION
# =========================
def extract_text_pdf(file):
    text = ""
    reader = PyPDF2.PdfReader(file)
    for page in reader.pages:
        extracted = page.extract_text()
        if extracted:
            text += extracted + "\n"
    return text


def extract_text_docx(file):
    doc = docx.Document(file)
    return "\n".join([p.text for p in doc.paragraphs])


# =========================
# BASIC INFO
# =========================
def extract_email(text):
    emails = re.findall(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}", text)
    return emails[0] if emails else "Not found"


def extract_phone(text):
    phones = re.findall(r"(?:\+91[- ]?)?[6-9]\d{9}", text)
    return phones[0] if phones else "Not found"


# =========================
# SKILL DATABASE
# =========================
SKILLS_DB = {
    "CSE": ['Python','Java','C++','SQL','Machine Learning','HTML','CSS','JavaScript','DBMS','Git'],
    "ECE": ['VLSI','Embedded Systems','Microcontrollers','Arduino','MATLAB','Verilog','VHDL','DSP','IoT'],
    "EEE": ['Power Systems','Control Systems','Electrical Machines','PLC','SCADA'],
    "Mechanical": ['AutoCAD','SolidWorks','CATIA','ANSYS','Thermodynamics'],
    "Civil": ['STAAD Pro','ETABS','Revit','Surveying','Structural Analysis'],
    "Chemical": ['Process Design','Heat Transfer','Mass Transfer','Aspen Plus'],
    "Biotech": ['Bioinformatics','Genomics','PCR','Cell Culture']
}

ALL_SKILLS = sorted(set(skill for skills in SKILLS_DB.values() for skill in skills))


def extract_skills(text):
    found = []
    lower_text = text.lower()
    for skill in ALL_SKILLS:
        if skill.lower() in lower_text:
            found.append(skill)
    return sorted(set(found))


# =========================
# DEPARTMENT DETECTION
# =========================
def detect_department(resume_skills):
    best_dept = "Unknown"
    best_score = 0
    dept_breakdown = {}

    for dept, dept_skills in SKILLS_DB.items():
        matched = len([s for s in dept_skills if s in resume_skills])
        percent = (matched / len(dept_skills)) * 100 if dept_skills else 0
        dept_breakdown[dept] = round(percent, 2)

        if percent > best_score:
            best_score = percent
            best_dept = dept

    return best_dept, best_score, dept_breakdown


def analyze_skill_gap(resume_skills, dept):
    dept_skills = SKILLS_DB.get(dept, [])
    matched = [s for s in dept_skills if s in resume_skills]
    missing = [s for s in dept_skills if s not in resume_skills]
    score = round((len(matched) / len(dept_skills)) * 100, 2) if dept_skills else 0
    return score, matched, missing


# =========================
# EXTRA SMART FEATURES
# =========================
def extract_sections(text):
    sections = {"Education": [], "Projects": [], "Experience": []}
    for line in text.split("\n"):
        low = line.lower()
        if "education" in low:
            sections["Education"].append(line)
        elif "project" in low:
            sections["Projects"].append(line)
        elif "experience" in low or "internship" in low:
            sections["Experience"].append(line)
    return sections


def placement_readiness(skills, missing):
    score = len(skills) * 5 - len(missing) * 3
    return max(0, min(100, score))


def suggest_roles(dept):
    role_map = {
        "CSE": ["Software Developer", "ML Engineer"],
        "ECE": ["VLSI Engineer", "Embedded Engineer"],
        "EEE": ["PLC Engineer", "Power Engineer"],
        "Mechanical": ["CAD Design Engineer"],
        "Civil": ["Structural Engineer"],
        "Chemical": ["Process Engineer"],
        "Biotech": ["Bioinformatics Analyst"]
    }
    return role_map.get(dept, ["Engineer"])


# =========================
# CHARTS
# =========================
def plot_bar(matched, missing):
    fig, ax = plt.subplots(figsize=(6, 4))
    ax.bar(['Matched', 'Missing'], [len(matched), len(missing)])
    ax.set_title('Skill Gap Analysis')
    st.pyplot(fig)


def plot_pie(matched, missing):
    fig, ax = plt.subplots(figsize=(5, 5))
    ax.pie([max(len(matched), 0.01), max(len(missing), 0.01)], labels=['Matched', 'Missing'], autopct='%1.1f%%')
    st.pyplot(fig)


# =========================
# PDF REPORT
# =========================
def generate_pdf(email, phone, dept, detected_score, skills, score, matched, missing, roles):
    pdf = FPDF()
    pdf.set_auto_page_break(auto=True, margin=15)
    pdf.add_page()

    # Header
    pdf.set_font("Arial", 'B', 20)
    pdf.cell(0, 12, "AI Resume Analysis Report", ln=True, align='C')
    pdf.set_font("Arial", 'I', 11)
    pdf.cell(0, 8, "Premium ATS-style Placement Intelligence Summary", ln=True, align='C')
    pdf.ln(8)

    # Candidate summary
    pdf.set_font("Arial", 'B', 14)
    pdf.cell(0, 8, "Candidate Summary", ln=True)
    pdf.set_font("Arial", '', 12)
    pdf.cell(0, 8, f"Email: {email}", ln=True)
    pdf.cell(0, 8, f"Phone: {phone}", ln=True)
    pdf.cell(0, 8, f"Detected Department: {dept}", ln=True)
    pdf.cell(0, 8, f"Department Confidence: {detected_score}%", ln=True)
    pdf.cell(0, 8, f"ATS Readiness Score: {score}%", ln=True)
    pdf.ln(4)

    # Skills overview
    pdf.set_font("Arial", 'B', 14)
    pdf.cell(0, 8, "Skills Overview", ln=True)
    pdf.set_font("Arial", '', 11)
    pdf.multi_cell(0, 7, "Detected Skills: " + (", ".join(skills) if skills else "No technical skills detected"))
    pdf.multi_cell(0, 7, "Matched Department Skills: " + (", ".join(matched) if matched else "None"))
    pdf.multi_cell(0, 7, "Recommended Missing Skills: " + (", ".join(missing) if missing else "No major missing skills"))
    pdf.ln(3)

    # Career recommendations
    pdf.set_font("Arial", 'B', 14)
    pdf.cell(0, 8, "Suggested Career Roles", ln=True)
    pdf.set_font("Arial", '', 11)
    pdf.multi_cell(0, 7, ", ".join(roles))
    pdf.ln(3)

    # Recommendation paragraph
    pdf.set_font("Arial", 'B', 14)
    pdf.cell(0, 8, "Professional Recommendation", ln=True)
    pdf.set_font("Arial", '', 11)
    recommendation = (
        f"The candidate profile is strongly aligned with {dept}. "
        f"To improve placement opportunities, focus on strengthening the missing skills listed above. "
        f"The current ATS readiness score of {score}% suggests {'strong' if score >= 75 else 'moderate' if score >= 50 else 'early-stage'} job readiness. "
        f"Recommended target roles include {', '.join(roles)}."
    )
    pdf.multi_cell(0, 7, recommendation)

    # Footer note
    pdf.ln(5)
    pdf.set_font("Arial", 'I', 10)
    pdf.multi_cell(0, 6, "Generated by Professional Smart Resume Analyzer V2")

    return pdf.output(dest='S').encode('latin-1')


# =========================
# UI
# =========================
st.title("🚀 Professional Smart Resume Analyzer")
st.caption("Automatic department detection + premium ATS-style placement intelligence report")

st.sidebar.header("Upload Resume")
file = st.sidebar.file_uploader("PDF / DOCX", type=["pdf", "docx"])

if file:
    text = extract_text_pdf(file) if file.name.endswith('.pdf') else extract_text_docx(file)

    email = extract_email(text)
    phone = extract_phone(text)
    skills = extract_skills(text)

    dept, detected_score, dept_breakdown = detect_department(skills)
    score, matched, missing = analyze_skill_gap(skills, dept)
    readiness = placement_readiness(skills, missing)
    roles = suggest_roles(dept)
    sections = extract_sections(text)

    st.subheader(f"🎓 Detected Department: {dept}")
    st.progress(score / 100)
    st.success(f"Department Readiness Score: {score}% | Placement Score: {readiness}%")

    col1, col2 = st.columns(2)
    with col1:
        st.metric("📧 Email", email)
        st.metric("📱 Phone", phone)
        st.info("🧠 Skills: " + (", ".join(skills) if skills else "None"))
    with col2:
        st.write("### 💼 Suggested Roles")
        st.write(", ".join(roles))
        st.write("### 🏢 Department Confidence")
        for d, s in sorted(dept_breakdown.items(), key=lambda x: x[1], reverse=True):
            st.write(f"**{d}**: {s}%")

    if missing:
        st.warning("⚠ Recommended Skills: " + ", ".join(missing))

    st.subheader("📊 Visual Analysis")
    plot_bar(matched, missing)
    plot_pie(matched, missing)

    pdf = generate_pdf(email, phone, dept, detected_score, skills, score, matched, missing, roles)
    st.download_button("📥 Download Professional PDF Report", pdf, "professional_resume_report.pdf")


# =========================
# ULTIMATE V3 FEATURES (Integrated)
# =========================

def ats_score_breakdown(skills, sections, missing):
    skills_score = min(len(skills) * 3, 30)
    projects_score = 25 if sections.get("Projects") else 5
    exp_score = 20 if sections.get("Experience") else 5
    edu_score = 15 if sections.get("Education") else 5
    cert_score = 10 if "certification" in " ".join(sum(sections.values(), [])).lower() else 2
    total = min(100, skills_score + projects_score + exp_score + edu_score + cert_score - len(missing))
    return total, {
        "Skills": skills_score,
        "Projects": projects_score,
        "Experience": exp_score,
        "Education": edu_score,
        "Certifications": cert_score
    }


def generate_interview_questions(dept, skills):
    base = {
        "CSE": ["Explain your best coding project.", "What is DBMS normalization?"],
        "ECE": ["Explain your embedded/VLSI project.", "What is the use of DSP?"],
        "Mechanical": ["Explain your CAD workflow.", "How does thermodynamics apply in your project?"],
    }
    return base.get(dept, ["Explain your strongest project experience."])


def company_fit_predictor(skills, company="TCS"):
    company_skills = {
        "TCS": ["Python", "SQL", "Communication"],
        "Infosys": ["Java", "DBMS", "Git"],
        "Google": ["Algorithms", "System Design", "Python"]
    }
    req = company_skills.get(company, [])
    matched = [s for s in req if s in skills]
    score = round((len(matched) / len(req)) * 100, 2) if req else 0
    missing = [s for s in req if s not in matched]
    return score, missing


def improvement_roadmap(missing):
    weeks = []
    for i, skill in enumerate(missing[:4], 1):
        weeks.append(f"Week {i}: Learn {skill}")
    return weeks
