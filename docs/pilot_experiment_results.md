# Pilot Experiment: MedGemma on 5 Fundus Images

## Setup
- **Model:** google/medgemma-4b-it (bfloat16, greedy decoding, `do_sample=False`)
- **Hardware:** Kaggle, GPU T4 x2
- **System prompt:** "You are an expert ophthalmologist."
- **Images:** one image per ICDR grade from the balanced sample (`random_state=42`)
  - Grade 0: 4095_right
  - Grade 1: 41804_left
  - Grade 2: 4698_right
  - Grade 3: 99_right
  - Grade 4: 9598_right
- **Purpose:** check that the pipeline works, measure generation time, and refine the structured template before the full run.

## Runs
1. **Zero-shot** (max_new_tokens = 400): "Describe the findings in this retinal fundus image and assess the diabetic retinopathy severity."
2. **Structured template v1** (max_new_tokens = 350): fixed output format + ICDR grade definitions; FINDINGS list had 5 items (microaneurysms, hemorrhages, hard exudates, cotton wool spots, neovascularization).
3. **Structured template v2 (final)**: added two findings (*Venous beading or IRMA*, *Vitreous or preretinal hemorrhage*) and changed RECOMMENDATION to "one actionable follow-up sentence". The prompt was frozen after this change.

## Results per image

**Grade 0 (4095_right)**
- Zero-shot: "No obvious signs of diabetic retinopathy" (correct), 36.3 s
- Structured v1: grade 0 (correct), 11.9 s
- Structured v2: grade 0 (correct), 15.1 s

**Grade 1 (41804_left)**
- Zero-shot: "Mild to Moderate" (range, not a single grade), 36.6 s, output truncated at the token limit
- Structured v1: grade 1 (correct), 13.6 s
- Structured v2: grade 1 (correct), 15.3 s

**Grade 2 (4698_right)**
- Zero-shot: "Mild to Moderate" (range, not a single grade), 33.9 s
- Structured v1: grade 2 (correct), 14.1 s
- Structured v2: grade 2 (correct), 15.9 s

**Grade 3 (99_right)**
- Zero-shot: "Severe" (correct), 36.7 s
- Structured v1: grade 2 (off by one), 14.7 s
- Structured v2: grade 3 (correct), 16.5 s

**Grade 4 (9598_right)**
- Zero-shot: "Severe" (incorrect; also mentioned vitreous hemorrhage, a sign of grade 4), 36.9 s, output truncated at the token limit
- Structured v1: grade 4 (correct), 12.9 s
- Structured v2: grade 4 (correct), 17.2 s

## Summary
- **Exact grade:** zero-shot 2/5, structured v1 4/5, structured v2 5/5
- **Average time per image:** zero-shot ~36 s, structured v1 ~13 s, structured v2 ~16 s
- **Format compliance:** structured v1 and v2 followed the template in 5/5 cases

## Observations
- **Zero-shot gives ranges** ("Mild to Moderate") instead of one grade, which makes automatic evaluation hard. This motivated the structured template.
- **Zero-shot outputs are long** with generic disclaimers and recommendations; 2/5 were cut off at the 400-token limit.
- **Internal inconsistency (zero-shot, grade 4 image):** the report mentioned vitreous hemorrhage (a proliferative sign) but concluded "Severe".
- **Possible hallucination (zero-shot, grade 1 image):** the report listed hemorrhages and cotton wool spots, which are signs of a higher grade than the label.
- **Template gap (structured v1):** the FINDINGS list had no field for venous beading/IRMA or vitreous hemorrhage, so the model could not justify grades 3 and 4. Adding these fields in v2 fixed the grade 3 case.
- **Remaining inconsistency (structured v2, grade 4 image):** grade 4 was assigned while *Neovascularization* and *Vitreous hemorrhage* were both marked Absent, and the SUMMARY mentioned neovascularization although the FINDINGS list said Absent. Instead of further tuning the prompt on the same 5 images (risk of overfitting the prompt to the pilot set), this was turned into an evaluation metric (internal consistency).
- **Clinically weak recommendation (structured v2, grade 4 image):** "Schedule a follow-up appointment" instead of urgent referral.

## Raw outputs

### Structured v2 (final prompt)

**4095_right | true grade 0 | 15.1 s**
```
IMAGE QUALITY: Adequate
FINDINGS:
- Microaneurysms: Absent
- Hemorrhages: Absent
- Hard exudates: Absent
- Cotton wool spots: Absent
- Venous beading or IRMA: Absent
- Neovascularization: Absent
- Vitreous or preretinal hemorrhage: Absent
SEVERITY GRADE: 0
SEVERITY LABEL: No DR
RECOMMENDATION: Schedule a follow-up appointment in 6 months.
SUMMARY: The fundus image shows no signs of diabetic retinopathy. The patient should be monitored regularly to detect any changes.
```

**41804_left | true grade 1 | 15.3 s**
```
IMAGE QUALITY: Adequate
FINDINGS:
- Microaneurysms: Present
- Hemorrhages: Absent
- Hard exudates: Absent
- Cotton wool spots: Absent
- Venous beading or IRMA: Absent
- Neovascularization: Absent
- Vitreous or preretinal hemorrhage: Absent
SEVERITY GRADE: 1
SEVERITY LABEL: Mild NPDR
RECOMMENDATION: Follow up in 6 months to monitor for progression.
SUMMARY: The image shows microaneurysms, indicating mild non-proliferative diabetic retinopathy. No other significant findings are present.
```

**4698_right | true grade 2 | 15.9 s**
```
IMAGE QUALITY: Adequate
FINDINGS:
- Microaneurysms: Present
- Hemorrhages: Present
- Hard exudates: Absent
- Cotton wool spots: Absent
- Venous beading or IRMA: Absent
- Neovascularization: Absent
- Vitreous or preretinal hemorrhage: Absent
SEVERITY GRADE: 2
SEVERITY LABEL: Moderate NPDR
RECOMMENDATION: Follow up in 6 months to monitor for progression.
SUMMARY: The image shows moderate non-proliferative diabetic retinopathy with microaneurysms and hemorrhages. No signs of severe features or neovascularization are present.
```

**99_right | true grade 3 | 16.5 s**
```
IMAGE QUALITY: Adequate
FINDINGS:
- Microaneurysms: Present
- Hemorrhages: Present
- Hard exudates: Present
- Cotton wool spots: Present
- Venous beading or IRMA: Present
- Neovascularization: Absent
- Vitreous or preretinal hemorrhage: Absent
SEVERITY GRADE: 3
SEVERITY LABEL: Severe NPDR
RECOMMENDATION: Follow up in 3 months to monitor for progression.
SUMMARY: The image shows significant retinal findings, including microaneurysms, hemorrhages, hard exudates, cotton wool spots, and venous beading. This indicates a high level of diabetic retinopathy.
```

**9598_right | true grade 4 | 17.2 s**
```
IMAGE QUALITY: Poor
FINDINGS:
- Microaneurysms: Present
- Hemorrhages: Present
- Hard exudates: Present
- Cotton wool spots: Present
- Venous beading or IRMA: Present
- Neovascularization: Absent
- Vitreous or preretinal hemorrhage: Absent
SEVERITY GRADE: 4
SEVERITY LABEL: Proliferative DR
RECOMMENDATION: Schedule a follow-up appointment to assess for progression and treatment options.
SUMMARY: The image shows significant retinal findings, including microaneurysms, hemorrhages, hard exudates, cotton wool spots, venous beading, and neovascularization. This indicates a high level of diabetic retinopathy.
```

### Structured v1 (before the template fix)

**4095_right | true grade 0 | 11.9 s**
```
IMAGE QUALITY: Adequate
FINDINGS:
- Microaneurysms: Absent
- Hemorrhages: Absent
- Hard exudates: Absent
- Cotton wool spots: Absent
- Neovascularization: Absent
SEVERITY GRADE: 0
SEVERITY LABEL: No DR
RECOMMENDATION: No diabetic retinopathy is present.
SUMMARY: The image shows a normal retina with no signs of diabetic retinopathy.
```

**41804_left | true grade 1 | 13.6 s**
```
IMAGE QUALITY: Adequate
FINDINGS:
- Microaneurysms: Present
- Hemorrhages: Absent
- Hard exudates: Absent
- Cotton wool spots: Absent
- Neovascularization: Absent
SEVERITY GRADE: 1
SEVERITY LABEL: Mild NPDR
RECOMMENDATION: Follow up with a comprehensive dilated fundus exam in 6 months.
SUMMARY: The image shows microaneurysms, indicating mild non-proliferative diabetic retinopathy. No other significant findings are present.
```

**4698_right | true grade 2 | 14.1 s**
```
IMAGE QUALITY: Adequate
FINDINGS:
- Microaneurysms: Present
- Hemorrhages: Present
- Hard exudates: Absent
- Cotton wool spots: Absent
- Neovascularization: Absent
SEVERITY GRADE: 2
SEVERITY LABEL: Moderate NPDR
RECOMMENDATION: Follow up with an ophthalmologist for further evaluation and management.
SUMMARY: The image shows moderate non-proliferative diabetic retinopathy with microaneurysms and hemorrhages. Further evaluation is needed to determine the best course of treatment.
```

**99_right | true grade 3 | 14.7 s**
```
IMAGE QUALITY: Adequate
FINDINGS:
- Microaneurysms: Present
- Hemorrhages: Present
- Hard exudates: Present
- Cotton wool spots: Present
- Neovascularization: Absent
SEVERITY GRADE: 2
SEVERITY LABEL: Moderate NPDR
RECOMMENDATION: Follow up with an ophthalmologist for further evaluation and management.
SUMMARY: The image shows moderate non-proliferative diabetic retinopathy with microaneurysms, hemorrhages, hard exudates, and cotton wool spots. This indicates a significant level of diabetic retinopathy.
```

**9598_right | true grade 4 | 12.9 s**
```
IMAGE QUALITY: Poor
FINDINGS:
- Microaneurysms: Present
- Hemorrhages: Present
- Hard exudates: Present
- Cotton wool spots: Present
- Neovascularization: Absent
SEVERITY GRADE: 4
SEVERITY LABEL: Proliferative DR
RECOMMENDATION: The patient has proliferative diabetic retinopathy.
SUMMARY: The image shows extensive retinal findings consistent with proliferative diabetic retinopathy. Further evaluation and management are required.
```

### Zero-shot (key statements; full outputs were long, two were cut at the token limit)

- **4095_right | true grade 0 | 36.3 s:** noted the image was dark and off-center; concluded "No obvious signs of diabetic retinopathy are visible"; recommended a dilated fundus exam; ended with a disclaimer.
- **41804_left | true grade 1 | 36.6 s:** listed attenuated vessels, subtle edema, microaneurysms, cotton wool spots and hemorrhages; concluded "Mild to Moderate"; output truncated.
- **4698_right | true grade 2 | 33.9 s:** listed microaneurysms and subtle retinal thickening, normal macula and optic disc; concluded "Mild to Moderate"; ended with a disclaimer.
- **99_right | true grade 3 | 36.7 s:** listed macular edema, microaneurysms, hemorrhages, cotton wool spots and vitreous hemorrhage; concluded "Severe".
- **9598_right | true grade 4 | 36.9 s:** listed extensive hemorrhages, cotton wool spots, macular edema, microaneurysms and vitreous hemorrhage; described the image as hazy; concluded "Severe"; output truncated.
