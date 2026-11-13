# ScamShield: AI-Powered Multi-Signal Scam & Phishing Detection System
**Applied Cybersecurity Final Year Project — Complete Implementation & Defense Guide**

---

## 1. Executive Summary & Abstract
**ScamShield** is an applied cybersecurity multi-modal detection system that fuses optical character recognition (OCR), natural language processing (NLP), URL-based machine learning classification, and threat intelligence blocklist lookups into a unified threat detection framework. 

Rather than relying on isolated detection mechanisms (which can be bypassed when a malicious message uses a clean domain or subtle phrasing), ScamShield's core original contribution is its **Risk Fusion Engine**. The Fusion Engine treats each subsystem output as a weak signal and applies weighted signal fusion and a trained meta-classifier (Logistic Regression) to calculate a final **Unified Risk Score (0–100)**, risk level classification (`HIGH RISK`, `MEDIUM RISK`, `LOW RISK`), threat category, and explicit evidence trace.

---

## 2. System Architecture

```
                                  +-----------------------+
                                  | Screenshot / Text Input|
                                  +-----------+-----------+
                                              |
                                              v
                                  +-----------------------+
                                  |    OCR Extraction     |
                                  |  (EasyOCR/Tesseract)  |
                                  +-----------+-----------+
                                              |
            +---------------------------------+---------------------------------+
            |                                 |                                 |
            v                                 v                                 v
+-----------------------+         +-----------------------+         +-----------------------+
|  NLP Scam Text Model  |         |   URL ML Classifier   |         | Rule-Based Social Eng |
| TF-IDF + Logistic Reg |         |  RF (16 Lexical Feat) |         | Pattern & Threat RegEx|
+-----------+-----------+         +-----------+-----------+         +-----------+-----------+
            |                                 |                                 |
            |                                 v                                 |
            |                     +-----------------------+                     |
            |                     | Known Threat Intel DB |                     |
            |                     | (local / PhishTank)   |                     |
            |                     +-----------+-----------+                     |
            |                                 |                                 |
            +---------------------------------+---------------------------------+
                                              |
                                              v
                                  +-----------------------+
                                  |  Risk Fusion Engine   |
                                  | Weighted / Meta-Model |
                                  +-----------+-----------+
                                              |
                                              v
                                  +-----------------------+
                                  | Streamlit Dashboard UI |
                                  | Score, Category & Evid|
                                  +-----------------------+
```

---

## 3. Project Structure

```
scamshield/
├── README.md                      # Comprehensive project & viva documentation
├── requirements.txt               # Dependencies
├── data/                          # Datasets and training samples
├── models/                        # Saved ML models & vectorizers
│   ├── url_classifier.joblib      # Random Forest URL Phishing Model & Scaler
│   ├── text_classifier.joblib     # TF-IDF + Logistic Regression NLP Model
│   └── fusion_meta_model.joblib   # Risk Fusion Meta-Classifier
├── src/
│   ├── __init__.py
│   ├── ocr_engine.py              # EasyOCR / PyTesseract OCR & URL Regex Extractor
│   ├── url_classifier.py          # 16-Feature Lexical URL Phishing Classifier
│   ├── nlp_classifier.py          # Multilingual NLP Scam Text Classifier (English + Hinglish)
│   ├── social_engineering.py      # Rule-based Urgency, Bank Impersonation & OTP Detector
│   ├── scam_category.py           # Evidence-based category rules; not a trained multi-class model
│   ├── threat_intel.py            # Local threat blocklist & optional PhishTank API
│   ├── risk_fusion.py             # Adaptive risk fusion and verdict assembly
│   ├── explainability.py          # LIME explanations for existing model scores
│   ├── qr_decoder.py              # Local OpenCV QR decoding
│   ├── brand_impersonation.py     # Trusted-brand/domain mismatch checks
│   └── train_models.py            # Model Training & Evaluation Suite
├── config/trusted_brands.json     # Configurable brand aliases and official domains
├── app.py                         # Streamlit Interactive Web Application
└── test_*.py                      # Unit and integration test suites
```

---

## 4. Subsystem & Original Contribution Breakdown

### A. Subsystems (Reused & Fine-Tuned ML Pipelines)
1. **OCR Text Engine (`src/ocr_engine.py`)**:
   - Multi-stage OCR fallback supporting EasyOCR (English + Hindi), PyTesseract, and regex URL parsing.
2. **Multilingual NLP Scam Classifier (`src/nlp_classifier.py`)**:
   - Tokenizes and cleans English and Hinglish text.
   - Fits TF-IDF vectorizer (1-2 n-grams) with Logistic Regression classifier.
3. **URL Phishing ML Classifier (`src/url_classifier.py`)**:
   - Extracts 16 structural features from URLs: URL length, hostname length, dot/hyphen count, IP address usage, HTTPS flag, subdomain count, shortening service usage (e.g. `bit.ly`), suspicious keyword count (`login`, `verify`, `bank`, `kyc`, etc.), digit ratio, hyphen prefix/suffix.
   - Classifies using Random Forest Classifier (100 estimators).
4. **Known Threat Intelligence (`src/threat_intel.py`)**:
   - Checks a local ScamShield blocklist and can query PhishTank when configured with an API key. Google Safe Browsing is not integrated.
5. **Social Engineering Detector (`src/social_engineering.py`)**:
   - Rule-based pattern matching for psychological tactics: artificial urgency, bank impersonation (SBI, HDFC, Paytm, YONO), service disconnection threats (electricity bill), credential/OTP requests.

### B. Adaptive Risk Fusion (`src/risk_fusion.py`)
The fusion backend combines available detector outputs into an explainable 0–100 risk score. It uses text scam probability, the maximum URL phishing score, social-engineering rules, threat-intelligence matches, and—when a result is available—brand-impersonation and QR-destination scores.

- **Weighted Signal Fusion Formula**:
  $$R = 100 \cdot \frac{\sum_{i \in A} w_i S_i}{\sum_{i \in A} w_i} + \text{SynergyBoost}$$
  Here, $A$ includes only signals with real detector results. Weights are configurable through `RiskFusionEngine(weights=...)`; weights are renormalized when signals are missing. Strong independent signals can add a 15-point consensus boost and a 25-point confirmed-threat boost.
- **Risk Categories**: Low (0–24), Medium (25–49), High (50–74), and Critical (75–100). The result also retains the legacy `risk_level` field for existing callers.
- **Optional signals**: Brand impersonation uses a rules-based comparison of named brands against linked domains. QR-decoded URLs are analyzed using the existing URL and threat-intelligence detectors and are included in the unified URL signal. Missing detections are shown as unavailable, not treated as clean predictions.
- **Trained Meta-Classifier**: The existing Logistic Regression model can be included as an additional weighted signal when its complete four-signal input vector is available.

### C. LIME Explanations
The **LIME Explanations** tab explains the existing message and URL classifier scores by perturbing input text and querying each classifier's actual `predict_proba` method. It shows influential message words and URL components, the exact score being explained, LIME's local fidelity (R²), and measured URL structure features. These are local approximations, not causal proof. LIME does not train or replace ScamShield's models. Install it with the other project dependencies using `python -m pip install -r requirements.txt`.

### D. QR Code Analysis
The screenshot uploader uses OpenCV to decode one or more QR codes locally from uploaded images. Decoded text is included in the existing NLP analysis; decoded URLs go through the existing URL phishing classifier and threat-intelligence lookup. QR payloads and URL scores are displayed as text, and decoded links are never opened. Invalid images show an upload error; valid images without QR codes continue through ordinary OCR analysis.

### E. Brand Impersonation Detection
`config/trusted_brands.json` is the editable allowlist of brand aliases and official domains. `BrandImpersonationDetector` checks official-domain mismatches, brand names embedded in unapproved registered domains, brand-like hostname spellings, and brand domains placed in non-official subdomain chains. A brand mention paired with a sensitive request but no URL produces only a limited caution signal. It returns a 0–1 risk signal with evidence and an assessment, which the Adaptive Risk Fusion scales and includes in its 0–100 score. Similarity-only matches receive a lower caution score and are explicitly not treated as proof; only actual allowlist matches are treated as recognized official domains. Supply a custom mapping or JSON path to `BrandImpersonationDetector` to override the defaults.

### F. English, Hindi, and Hinglish Text Processing
The existing TF-IDF/Logistic Regression classifier is retained. The NLP pipeline identifies English, Hindi-script, Romanized Hinglish, and mixed-script inputs, normalizes common Hindi/Hinglish words and spelling variants into the existing model's vocabulary, and passes the normalized text through both its saved classifier and established heuristic blend. The UI shows the detected language and, for non-English inputs, the normalized text used for scoring. No MuRIL or IndicBERT weights are present in this project; transformer inference is not claimed or substituted for the saved model.

### G. Scam Category Output
`category_prediction` reports Phishing, OTP Fraud, Banking Fraud, UPI Fraud, Job Scams, Investment Scams, Lottery Scams, Impersonation, or Unknown based on explicit evidence rules. Its `confidence` is rule-evidence strength, not a calibrated model probability. The existing binary training datasets do not provide labels for these categories, so ScamShield does not claim trained multi-class classification. The older `scam_category` field remains for compatibility and returns `Unknown` when its legacy rules do not match.

---

## 5. Evaluation

`python -m src.train_models` computes and prints held-out metrics from the local datasets. No fixed accuracy figures are asserted here: results depend on the dataset contents, split, and model artifact versions. The current datasets are binary-label datasets and do not validate the category rules or guarantee real-world performance.

---

## 6. How to Run the Project

### Installation
```bash
cd scamshield
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

### Train & Evaluate ML Models
```bash
python -m src.train_models
```

### Run Unit & Integration Test Suite
```bash
python -m unittest discover -v -p 'test_*.py'
```

### Launch Interactive Streamlit Web Application
```bash
streamlit run app.py
```

---

## 7. Viva Defense & Demonstration Checklist

When presenting this project during your Viva:
1. **Explain the Cybersecurity Need**: Emphasize that SMS phishing (Smishing) and WhatsApp scams exploit human trust. Single-signal tools (like basic browser extensions or standard SMS filters) miss multi-vector attacks.
2. **Highlight the Original Contribution**: Point out that while OCR, NLP, and Random Forest classifiers exist in open source, your **Risk Fusion Engine** is the novel contribution that unifies these weak signals into an explainable threat verdict.
3. **Demonstrate Indian Context (English + Hinglish)**: Show preset test cases for fake SBI KYC update alerts, electricity bill disconnection notices (in Hinglish), and Paytm wallet scams.
4. **Showcase Explainability**: Demonstrate how ScamShield not only gives a 0-100 risk score but explains *why* (e.g. "Matched Threat of Immediate Action ('account blocked')", "Suspicious domain structure", "High NLP scam score").
