# intel-task
import streamlit as st
import re
import pandas as pd


def extract_clauses(text):
    # Regular expression to find numbered clauses
    clause_pattern = r'(\d+)\.\s+([^.]+)'
    clauses = re.findall(clause_pattern, text)
    return {num: title.strip() for num, title in clauses}


def extract_subclauses(text, clause):
    # Find the text of the specific clause
    clause_text = re.findall(f'{clause}\\.[^\\d]+(?=\\d+\\.|$)', text, re.DOTALL)
    if not clause_text:
        return []

    # Extract bullet points or lettered subclauses
    subclause_pattern = r'([A-Z]\.|\-)\s*(.+?)(?=[A-Z]\.|\-|$)'
    subclauses = re.findall(subclause_pattern, clause_text[0], re.DOTALL)
    return [subclause.strip() for _, subclause in subclauses]


def analyze_contract(contract_text):
    clauses = extract_clauses(contract_text)

    analysis = {}
    for num, title in clauses.items():
        subclauses = extract_subclauses(contract_text, num)
        analysis[f"{num}. {title}"] = subclauses

    return analysis


def detect_deviations(analysis, template_analysis):
    deviations = {}
    for clause in template_analysis:
        if clause not in analysis:
            deviations[clause] = "Missing clause"
        else:
            template_subclauses = set(template_analysis[clause])
            actual_subclauses = set(analysis[clause])
            if template_subclauses != actual_subclauses:
                deviations[clause] = {
                    "Missing subclauses": list(template_subclauses - actual_subclauses),
                    "Additional subclauses": list(actual_subclauses - template_subclauses)
                }
    return deviations


def main():
    st.title("Business Contract Analyzer")

    # Template contract (based on the image provided)
    template_contract = """
    1. Services. Service Provider agrees to provide and Buyer agrees to purchase the following services for the specific projects described below:
    2. Purchase Price. Buyer will pay to Service Provider and for all obligations specified in this Agreement, if any, as the full and complete purchase price, the sum of ___________.
    3. Payment. Payment for the Services will be by __________, according to the following schedule:
    4. Delivery. Seller shall ship the Goods to Buyer on or before __________ at the following address:
    5. Risk of Loss. Title to and risk of loss of the Goods shall pass to Buyer upon delivery of the Goods to Buyer in accordance with this Agreement.
    6. Security Interest. Buyer hereby grants to Service Provider a security interest in any final products resulting from said services, until Buyer has paid Service Provider in full. Buyer shall sign and deliver any document needed to perfect the security interest that Service Provider reasonably requests.
    """

    # Analyze the template
    template_analysis = analyze_contract(template_contract)

    # File uploader for the actual contract
    uploaded_file = st.file_uploader("Choose a contract file", type="txt")

    if uploaded_file is not None:
        contract_text = uploaded_file.getvalue().decode("utf-8")
        st.text_area("Contract Text", contract_text, height=200)

        # Analyze the uploaded contract
        analysis = analyze_contract(contract_text)

        # Display the analysis
        st.subheader("Contract Analysis")
        for clause, subclauses in analysis.items():
            st.write(f"**{clause}**")
            for subclause in subclauses:
                st.write(f"- {subclause}")

        # Detect deviations
        deviations = detect_deviations(analysis, template_analysis)

        # Display deviations
        st.subheader("Deviations from Template")
        if deviations:
            for clause, deviation in deviations.items():
                st.write(f"**{clause}**")
                if isinstance(deviation, str):
                    st.write(deviation)
                else:
                    if deviation["Missing subclauses"]:
                        st.write("Missing subclauses:")
                        for subclause in deviation["Missing subclauses"]:
                            st.write(f"- {subclause}")
                    if deviation["Additional subclauses"]:
                        st.write("Additional subclauses:")
                        for subclause in deviation["Additional subclauses"]:
                            st.write(f"- {subclause}")
        else:
            st.write("No significant deviations detected.")


if __name__ == "__main__":
    main()
