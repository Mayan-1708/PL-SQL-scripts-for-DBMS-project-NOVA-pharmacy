-- Create database for NOVA Pharmacy Chain
CREATE DATABASE IF NOT EXISTS nova_pharmacy;
USE nova_pharmacy;

-- Patient table
CREATE TABLE Patient (
    aadharID VARCHAR(12) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    address VARCHAR(255) NOT NULL,
    age INT CHECK (age > 0),
    primaryPhysician VARCHAR(12) NOT NULL
);

-- Doctor table
CREATE TABLE Doctor (
    aadharID VARCHAR(12) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    specialty VARCHAR(100) NOT NULL,
    yearsOfExperience INT CHECK (yearsOfExperience >= 0)
);

-- Adding foreign key to Patient after Doctor table is created
ALTER TABLE Patient 
ADD CONSTRAINT fk_patient_doctor 
FOREIGN KEY (primaryPhysician) REFERENCES Doctor(aadharID);

-- Pharmaceutical Company table
CREATE TABLE PharmaceuticalCompany (
    name VARCHAR(100) PRIMARY KEY,
    phoneNumber VARCHAR(15) NOT NULL
);

-- Pharmacy table
CREATE TABLE Pharmacy (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    address VARCHAR(255) NOT NULL,
    phone VARCHAR(15) NOT NULL
);

-- Drug table with pharmaceutical company foreign key
CREATE TABLE Drug (
    id INT AUTO_INCREMENT PRIMARY KEY,
    tradeName VARCHAR(100) NOT NULL,
    formula VARCHAR(255) NOT NULL,
    companyName VARCHAR(100) NOT NULL,
    CONSTRAINT fk_drug_company FOREIGN KEY (companyName) REFERENCES PharmaceuticalCompany(name) ON DELETE CASCADE,
    UNIQUE KEY unique_tradename_company (tradeName, companyName)
);

-- Contract table for pharmaceutical companies and pharmacies
CREATE TABLE Contract (
    id INT AUTO_INCREMENT PRIMARY KEY,
    pharmacyID INT NOT NULL,
    companyName VARCHAR(100) NOT NULL,
    startDate DATE NOT NULL,
    endDate DATE NOT NULL,
    contractContent TEXT NOT NULL,
    supervisorName VARCHAR(100) NOT NULL,
    CONSTRAINT fk_contract_pharmacy FOREIGN KEY (pharmacyID) REFERENCES Pharmacy(id),
    CONSTRAINT fk_contract_company FOREIGN KEY (companyName) REFERENCES PharmaceuticalCompany(name),
    CHECK (endDate > startDate)
);

-- Pharmacy Drug table (junction table between Pharmacy and Drug)
CREATE TABLE PharmacyDrug (
    pharmacyID INT NOT NULL,
    drugID INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price > 0),
    stock INT NOT NULL DEFAULT 0 CHECK (stock >= 0),
    PRIMARY KEY (pharmacyID, drugID),
    CONSTRAINT fk_pharmacy_drug_pharmacy FOREIGN KEY (pharmacyID) REFERENCES Pharmacy(id),
    CONSTRAINT fk_pharmacy_drug_drug FOREIGN KEY (drugID) REFERENCES Drug(id)
);

-- Prescription table
CREATE TABLE Prescription (
    id INT AUTO_INCREMENT PRIMARY KEY,
    patientID VARCHAR(12) NOT NULL,
    doctorID VARCHAR(12) NOT NULL,
    prescriptionDate DATE NOT NULL,
    CONSTRAINT fk_prescription_patient FOREIGN KEY (patientID) REFERENCES Patient(aadharID),
    CONSTRAINT fk_prescription_doctor FOREIGN KEY (doctorID) REFERENCES Doctor(aadharID),
    UNIQUE KEY unique_patient_doctor_date (patientID, doctorID, prescriptionDate)
);

-- Prescription Detail table
CREATE TABLE PrescriptionDetail (
    prescriptionID INT NOT NULL,
    drugID INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (prescriptionID, drugID),
    CONSTRAINT fk_prescription_detail_prescription FOREIGN KEY (prescriptionID) REFERENCES Prescription(id),
    CONSTRAINT fk_prescription_detail_drug FOREIGN KEY (drugID) REFERENCES Drug(id)
);

-- Trigger to ensure doctor has at least one patient
DELIMITER //
CREATE TRIGGER check_doctor_has_patients BEFORE DELETE ON Patient
FOR EACH ROW
BEGIN
    DECLARE doctor_patient_count INT;
    
    SELECT COUNT(*) INTO doctor_patient_count 
    FROM Patient 
    WHERE primaryPhysician = OLD.primaryPhysician AND aadharID != OLD.aadharID;
    
    IF doctor_patient_count = 0 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Cannot delete patient: Doctor must have at least one patient';
    END IF;
END //
DELIMITER ;

-- Stored Procedure 1: Add a new pharmacy
DELIMITER //
CREATE PROCEDURE AddPharmacy(
    IN p_name VARCHAR(100),
    IN p_address VARCHAR(255),
    IN p_phone VARCHAR(15)
)
BEGIN
    INSERT INTO Pharmacy (name, address, phone) 
    VALUES (p_name, p_address, p_phone);
    SELECT LAST_INSERT_ID() AS pharmacy_id;
END //
DELIMITER ;

-- Stored Procedure 2: Add a new pharmaceutical company
DELIMITER //
CREATE PROCEDURE AddPharmaceuticalCompany(
    IN p_name VARCHAR(100),
    IN p_phone VARCHAR(15)
)
BEGIN
    INSERT INTO PharmaceuticalCompany (name, phoneNumber) 
    VALUES (p_name, p_phone);
END //
DELIMITER ;

-- Stored Procedure 3: Add a new patient
DELIMITER //
CREATE PROCEDURE AddPatient(
    IN p_aadharID VARCHAR(12),
    IN p_name VARCHAR(100),
    IN p_address VARCHAR(255),
    IN p_age INT,
    IN p_primaryPhysician VARCHAR(12)
)
BEGIN
    -- Check if doctor exists
    DECLARE doctor_exists INT;
    SELECT COUNT(*) INTO doctor_exists FROM Doctor WHERE aadharID = p_primaryPhysician;
    
    IF doctor_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Primary physician does not exist';
    ELSE
        INSERT INTO Patient (aadharID, name, address, age, primaryPhysician)
        VALUES (p_aadharID, p_name, p_address, p_age, p_primaryPhysician);
    END IF;
END //
DELIMITER ;

-- Stored Procedure 4: Add a new doctor
DELIMITER //
CREATE PROCEDURE AddDoctor(
    IN p_aadharID VARCHAR(12),
    IN p_name VARCHAR(100),
    IN p_specialty VARCHAR(100),
    IN p_yearsOfExperience INT
)
BEGIN
    INSERT INTO Doctor (aadharID, name, specialty, yearsOfExperience)
    VALUES (p_aadharID, p_name, p_specialty, p_yearsOfExperience);
END //
DELIMITER ;

-- Stored Procedure 5: Add a new drug
DELIMITER //
CREATE PROCEDURE AddDrug(
    IN p_tradeName VARCHAR(100),
    IN p_formula VARCHAR(255),
    IN p_companyName VARCHAR(100)
)
BEGIN
    -- Check if pharmaceutical company exists
    DECLARE company_exists INT;
    SELECT COUNT(*) INTO company_exists FROM PharmaceuticalCompany WHERE name = p_companyName;
    
    IF company_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Pharmaceutical company does not exist';
    ELSE
        INSERT INTO Drug (tradeName, formula, companyName)
        VALUES (p_tradeName, p_formula, p_companyName);
        SELECT LAST_INSERT_ID() AS drug_id;
    END IF;
END //
DELIMITER ;

-- Stored Procedure 6: Add drug to pharmacy with price
DELIMITER //
CREATE PROCEDURE AddDrugToPharmacy(
    IN p_pharmacyID INT,
    IN p_drugID INT,
    IN p_price DECIMAL(10, 2),
    IN p_initialStock INT
)
BEGIN
    -- Check if pharmacy and drug exist
    DECLARE pharmacy_exists INT;
    DECLARE drug_exists INT;
    
    SELECT COUNT(*) INTO pharmacy_exists FROM Pharmacy WHERE id = p_pharmacyID;
    SELECT COUNT(*) INTO drug_exists FROM Drug WHERE id = p_drugID;
    
    IF pharmacy_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Pharmacy does not exist';
    ELSEIF drug_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Drug does not exist';
    ELSE
        INSERT INTO PharmacyDrug (pharmacyID, drugID, price, stock)
        VALUES (p_pharmacyID, p_drugID, p_price, p_initialStock)
        ON DUPLICATE KEY UPDATE price = p_price, stock = p_initialStock;
    END IF;
END //
DELIMITER ;

-- Stored Procedure 7: Add a new contract
DELIMITER //
CREATE PROCEDURE AddContract(
    IN p_pharmacyID INT,
    IN p_companyName VARCHAR(100),
    IN p_startDate DATE,
    IN p_endDate DATE,
    IN p_contractContent TEXT,
    IN p_supervisorName VARCHAR(100)
)
BEGIN
    -- Check if pharmacy and company exist
    DECLARE pharmacy_exists INT;
    DECLARE company_exists INT;
    
    SELECT COUNT(*) INTO pharmacy_exists FROM Pharmacy WHERE id = p_pharmacyID;
    SELECT COUNT(*) INTO company_exists FROM PharmaceuticalCompany WHERE name = p_companyName;
    
    IF pharmacy_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Pharmacy does not exist';
    ELSEIF company_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Pharmaceutical company does not exist';
    ELSE
        INSERT INTO Contract (pharmacyID, companyName, startDate, endDate, contractContent, supervisorName)
        VALUES (p_pharmacyID, p_companyName, p_startDate, p_endDate, p_contractContent, p_supervisorName);
        SELECT LAST_INSERT_ID() AS contract_id;
    END IF;
END //
DELIMITER ;

-- Stored Procedure 8: Update contract supervisor
DELIMITER //
CREATE PROCEDURE UpdateContractSupervisor(
    IN p_contractID INT,
    IN p_newSupervisorName VARCHAR(100)
)
BEGIN
    UPDATE Contract
    SET supervisorName = p_newSupervisorName
    WHERE id = p_contractID;
    
    IF ROW_COUNT() = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Contract not found';
    END IF;
END //
DELIMITER ;

-- Stored Procedure 9: Add a new prescription
DELIMITER //
CREATE PROCEDURE AddPrescription(
    IN p_patientID VARCHAR(12),
    IN p_doctorID VARCHAR(12),
    IN p_prescriptionDate DATE
)
BEGIN
    -- Check if patient and doctor exist
    DECLARE patient_exists INT;
    DECLARE doctor_exists INT;
    DECLARE existing_prescription INT;
    
    SELECT COUNT(*) INTO patient_exists FROM Patient WHERE aadharID = p_patientID;
    SELECT COUNT(*) INTO doctor_exists FROM Doctor WHERE aadharID = p_doctorID;
    
    IF patient_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Patient does not exist';
    ELSEIF doctor_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Doctor does not exist';
    ELSE
        -- Check if a prescription already exists for this patient and doctor
        SELECT id INTO existing_prescription FROM Prescription 
        WHERE patientID = p_patientID AND doctorID = p_doctorID
        LIMIT 1;
        
        IF existing_prescription IS NOT NULL THEN
            -- Update existing prescription date if it's older
            UPDATE Prescription 
            SET prescriptionDate = p_prescriptionDate 
            WHERE id = existing_prescription AND prescriptionDate < p_prescriptionDate;
            
            -- Delete old prescription details to replace with new ones
            IF ROW_COUNT() > 0 THEN
                DELETE FROM PrescriptionDetail WHERE prescriptionID = existing_prescription;
            END IF;
            
            SELECT existing_prescription AS prescription_id;
        ELSE
            -- Create new prescription
            INSERT INTO Prescription (patientID, doctorID, prescriptionDate)
            VALUES (p_patientID, p_doctorID, p_prescriptionDate);
            SELECT LAST_INSERT_ID() AS prescription_id;
        END IF;
    END IF;
END //
DELIMITER ;

-- Stored Procedure 10: Add drug to prescription
DELIMITER //
CREATE PROCEDURE AddDrugToPrescription(
    IN p_prescriptionID INT,
    IN p_drugID INT,
    IN p_quantity INT
)
BEGIN
    -- Check if prescription and drug exist
    DECLARE prescription_exists INT;
    DECLARE drug_exists INT;
    
    SELECT COUNT(*) INTO prescription_exists FROM Prescription WHERE id = p_prescriptionID;
    SELECT COUNT(*) INTO drug_exists FROM Drug WHERE id = p_drugID;
    
    IF prescription_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Prescription does not exist';
    ELSEIF drug_exists = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Drug does not exist';
    ELSE
        INSERT INTO PrescriptionDetail (prescriptionID, drugID, quantity)
        VALUES (p_prescriptionID, p_drugID, p_quantity)
        ON DUPLICATE KEY UPDATE quantity = p_quantity;
    END IF;
END //
DELIMITER ;

-- Stored Procedure 11: Delete a pharmacy
DELIMITER //
CREATE PROCEDURE DeletePharmacy(
    IN p_pharmacyID INT
)
BEGIN
    -- Delete related records first
    DELETE FROM PharmacyDrug WHERE pharmacyID = p_pharmacyID;
    DELETE FROM Contract WHERE pharmacyID = p_pharmacyID;
    
    -- Then delete the pharmacy
    DELETE FROM Pharmacy WHERE id = p_pharmacyID;
    
    IF ROW_COUNT() = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Pharmacy not found';
    END IF;
END //
DELIMITER ;

-- Stored Procedure 12: Delete a pharmaceutical company
DELIMITER //
CREATE PROCEDURE DeletePharmaceuticalCompany(
    IN p_companyName VARCHAR(100)
)
BEGIN
    -- Drugs will be automatically deleted due to CASCADE
    DELETE FROM Contract WHERE companyName = p_companyName;
    DELETE FROM PharmaceuticalCompany WHERE name = p_companyName;
    
    IF ROW_COUNT() = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Pharmaceutical company not found';
    END IF;
END //
DELIMITER ;

-- Stored Procedure 13: Generate patient prescription report for a given period
DELIMITER //
CREATE PROCEDURE GetPatientPrescriptionsInPeriod(
    IN p_patientID VARCHAR(12),
    IN p_startDate DATE,
    IN p_endDate DATE
)
BEGIN
    SELECT 
        p.id AS prescription_id,
        p.prescriptionDate,
        d.name AS doctor_name,
        d.specialty AS doctor_specialty,
        drug.tradeName AS drug_name,
        drug.formula AS drug_formula,
        drug.companyName AS manufacturer,
        pd.quantity
    FROM 
        Prescription p
    JOIN 
        Doctor d ON p.doctorID = d.aadharID
    JOIN 
        PrescriptionDetail pd ON p.id = pd.prescriptionID
    JOIN 
        Drug drug ON pd.drugID = drug.id
    WHERE 
        p.patientID = p_patientID
        AND p.prescriptionDate BETWEEN p_startDate AND p_endDate
    ORDER BY 
        p.prescriptionDate DESC;
END //
DELIMITER ;

-- Stored Procedure 14: Print prescription details for a given patient and date
DELIMITER //
CREATE PROCEDURE GetPrescriptionDetails(
    IN p_patientID VARCHAR(12),
    IN p_date DATE
)
BEGIN
    SELECT 
        p.id AS prescription_id,
        pt.name AS patient_name,
        pt.age AS patient_age,
        pt.address AS patient_address,
        d.name AS doctor_name,
        d.specialty AS doctor_specialty,
        p.prescriptionDate,
        drug.tradeName AS drug_name,
        drug.formula AS drug_formula,
        drug.companyName AS manufacturer,
        pd.quantity
    FROM 
        Prescription p
    JOIN 
        Patient pt ON p.patientID = pt.aadharID
    JOIN 
        Doctor d ON p.doctorID = d.aadharID
    JOIN 
        PrescriptionDetail pd ON p.id = pd.prescriptionID
    JOIN 
        Drug drug ON pd.drugID = drug.id
    WHERE 
        p.patientID = p_patientID
        AND p.prescriptionDate = p_date
    ORDER BY 
        drug.tradeName;
END //
DELIMITER ;

-- Stored Procedure 15: Get details of drugs produced by a company
DELIMITER //
CREATE PROCEDURE GetCompanyDrugs(
    IN p_companyName VARCHAR(100)
)
BEGIN
    SELECT 
        id AS drug_id,
        tradeName,
        formula,
        (SELECT COUNT(*) FROM PharmacyDrug WHERE drugID = Drug.id) AS available_at_pharmacies_count
    FROM 
        Drug
    WHERE 
        companyName = p_companyName
    ORDER BY 
        tradeName;
END //
DELIMITER ;

-- Stored Procedure 16: Print pharmacy stock position
DELIMITER //
CREATE PROCEDURE GetPharmacyStock(
    IN p_pharmacyID INT
)
BEGIN
    SELECT 
        p.name AS pharmacy_name,
        p.address AS pharmacy_address,
        d.tradeName AS drug_name,
        d.formula AS drug_formula,
        d.companyName AS manufacturer,
        pd.price AS selling_price,
        pd.stock AS available_quantity
    FROM 
        Pharmacy p
    JOIN 
        PharmacyDrug pd ON p.id = pd.pharmacyID
    JOIN 
        Drug d ON pd.drugID = d.id
    WHERE 
        p.id = p_pharmacyID
    ORDER BY 
        d.tradeName;
END //
DELIMITER ;

-- Stored Procedure 17: Get pharmacy-pharmaceutical company contract details
DELIMITER //
CREATE PROCEDURE GetContractDetails(
    IN p_pharmacyID INT,
    IN p_companyName VARCHAR(100)
)
BEGIN
    SELECT 
        c.id AS contract_id,
        p.name AS pharmacy_name,
        p.address AS pharmacy_address,
        p.phone AS pharmacy_phone,
        pc.name AS company_name,
        pc.phoneNumber AS company_phone,
        c.startDate,
        c.endDate,
        c.supervisorName,
        c.contractContent
    FROM 
        Contract c
    JOIN 
        Pharmacy p ON c.pharmacyID = p.id
    JOIN 
        PharmaceuticalCompany pc ON c.companyName = pc.name
    WHERE 
        c.pharmacyID = p_pharmacyID
        AND c.companyName = p_companyName
    ORDER BY 
        c.endDate DESC;
END //
DELIMITER ;

-- Stored Procedure 18: Get patients list for a doctor
DELIMITER //
CREATE PROCEDURE GetDoctorPatients(
    IN p_doctorID VARCHAR(12)
)
BEGIN
    SELECT 
        p.aadharID,
        p.name,
        p.address,
        p.age,
        (SELECT COUNT(*) FROM Prescription WHERE doctorID = p_doctorID AND patientID = p.aadharID) AS prescription_count
    FROM 
        Patient p
    WHERE 
        p.primaryPhysician = p_doctorID
    ORDER BY 
        p.name;
END //
DELIMITER ;

-- Stored Procedure 19: Update drug stock at pharmacy
DELIMITER //
CREATE PROCEDURE UpdateDrugStock(
    IN p_pharmacyID INT,
    IN p_drugID INT,
    IN p_newStock INT
)
BEGIN
    UPDATE PharmacyDrug
    SET stock = p_newStock
    WHERE pharmacyID = p_pharmacyID AND drugID = p_drugID;
    
    IF ROW_COUNT() = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Drug not found in pharmacy inventory';
    END IF;
END //
DELIMITER ;

-- Stored Procedure 20: Update drug price at pharmacy
DELIMITER //
CREATE PROCEDURE UpdateDrugPrice(
    IN p_pharmacyID INT,
    IN p_drugID INT,
    IN p_newPrice DECIMAL(10, 2)
)
BEGIN
    UPDATE PharmacyDrug
    SET price = p_newPrice
    WHERE pharmacyID = p_pharmacyID AND drugID = p_drugID;
    
    IF ROW_COUNT() = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Drug not found in pharmacy inventory';
    END IF;
END //
DELIMITER ;



-- Stored Procedure: UpdatePatient  
DELIMITER //  
CREATE PROCEDURE UpdatePatient(  
    IN p_aadharID VARCHAR(12),  
    IN p_name VARCHAR(100),  
    IN p_address VARCHAR(255),  
    IN p_age INT,  
    IN p_primaryPhysician VARCHAR(12)  
)  
BEGIN  
    UPDATE Patient  
    SET name = p_name, address = p_address, age = p_age, primaryPhysician = p_primaryPhysician  
    WHERE aadharID = p_aadharID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Patient not found or no change detected';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: DeletePatient  
DELIMITER //  
CREATE PROCEDURE DeletePatient(  
    IN p_aadharID VARCHAR(12)  
)  
BEGIN  
    DELETE FROM Patient WHERE aadharID = p_aadharID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Patient not found';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: UpdateDoctor  
DELIMITER //  
CREATE PROCEDURE UpdateDoctor(  
    IN p_aadharID VARCHAR(12),  
    IN p_name VARCHAR(100),  
    IN p_specialty VARCHAR(100),  
    IN p_yearsOfExperience INT  
)  
BEGIN  
    UPDATE Doctor  
    SET name = p_name, specialty = p_specialty, yearsOfExperience = p_yearsOfExperience  
    WHERE aadharID = p_aadharID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Doctor not found or no change detected';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: DeleteDoctor  
DELIMITER //  
CREATE PROCEDURE DeleteDoctor(  
    IN p_aadharID VARCHAR(12)  
)  
BEGIN  
    DELETE FROM Doctor WHERE aadharID = p_aadharID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Doctor not found';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: UpdatePharmaceuticalCompany  
DELIMITER //  
CREATE PROCEDURE UpdatePharmaceuticalCompany(  
    IN p_name VARCHAR(100),  
    IN p_phone VARCHAR(15)  
)  
BEGIN  
    UPDATE PharmaceuticalCompany  
    SET phoneNumber = p_phone  
    WHERE name = p_name;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Pharmaceutical company not found or no change detected';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: UpdatePharmacy  
DELIMITER //  
CREATE PROCEDURE UpdatePharmacy(  
    IN p_pharmacyID INT,  
    IN p_name VARCHAR(100),  
    IN p_address VARCHAR(255),  
    IN p_phone VARCHAR(15)  
)  
BEGIN  
    UPDATE Pharmacy  
    SET name = p_name, address = p_address, phone = p_phone  
    WHERE id = p_pharmacyID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Pharmacy not found or no change detected';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: UpdateDrug  
DELIMITER //  
CREATE PROCEDURE UpdateDrug(  
    IN p_drugID INT,  
    IN p_tradeName VARCHAR(100),  
    IN p_formula VARCHAR(255),  
    IN p_companyName VARCHAR(100)  
)  
BEGIN  
    UPDATE Drug  
    SET tradeName = p_tradeName, formula = p_formula, companyName = p_companyName  
    WHERE id = p_drugID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Drug not found or no change detected';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: DeleteDrug  
DELIMITER //  
CREATE PROCEDURE DeleteDrug(  
    IN p_drugID INT  
)  
BEGIN  
    DELETE FROM Drug WHERE id = p_drugID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Drug not found';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: UpdateContract  
DELIMITER //  
CREATE PROCEDURE UpdateContract(  
    IN p_contractID INT,  
    IN p_startDate DATE,  
    IN p_endDate DATE,  
    IN p_contractContent TEXT,  
    IN p_supervisorName VARCHAR(100)  
)  
BEGIN  
    UPDATE Contract  
    SET startDate = p_startDate, endDate = p_endDate, contractContent = p_contractContent, supervisorName = p_supervisorName  
    WHERE id = p_contractID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Contract not found or no change detected';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: DeleteContract  
DELIMITER //  
CREATE PROCEDURE DeleteContract(  
    IN p_contractID INT  
)  
BEGIN  
    DELETE FROM Contract WHERE id = p_contractID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Contract not found';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: UpdatePharmacyDrug  
DELIMITER //  
CREATE PROCEDURE UpdatePharmacyDrug(  
    IN p_pharmacyID INT,  
    IN p_drugID INT,  
    IN p_price DECIMAL(10,2),  
    IN p_stock INT  
)  
BEGIN  
    UPDATE PharmacyDrug  
    SET price = p_price, stock = p_stock  
    WHERE pharmacyID = p_pharmacyID AND drugID = p_drugID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'PharmacyDrug entry not found or no change detected';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: DeletePharmacyDrug  
DELIMITER //  
CREATE PROCEDURE DeletePharmacyDrug(  
    IN p_pharmacyID INT,  
    IN p_drugID INT  
)  
BEGIN  
    DELETE FROM PharmacyDrug WHERE pharmacyID = p_pharmacyID AND drugID = p_drugID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'PharmacyDrug entry not found';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: UpdatePrescription  
DELIMITER //  
CREATE PROCEDURE UpdatePrescription(  
    IN p_prescriptionID INT,  
    IN p_patientID VARCHAR(12),  
    IN p_doctorID VARCHAR(12),  
    IN p_prescriptionDate DATE  
)  
BEGIN  
    UPDATE Prescription  
    SET patientID = p_patientID, doctorID = p_doctorID, prescriptionDate = p_prescriptionDate  
    WHERE id = p_prescriptionID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Prescription not found or no change detected';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: DeletePrescription  
DELIMITER //  
CREATE PROCEDURE DeletePrescription(  
    IN p_prescriptionID INT  
)  
BEGIN  
    -- Remove related details first  
    DELETE FROM PrescriptionDetail WHERE prescriptionID = p_prescriptionID;  
    DELETE FROM Prescription WHERE id = p_prescriptionID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Prescription not found';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: UpdatePrescriptionDetail  
DELIMITER //  
CREATE PROCEDURE UpdatePrescriptionDetail(  
    IN p_prescriptionID INT,  
    IN p_drugID INT,  
    IN p_quantity INT  
)  
BEGIN  
    UPDATE PrescriptionDetail  
    SET quantity = p_quantity  
    WHERE prescriptionID = p_prescriptionID AND drugID = p_drugID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'PrescriptionDetail entry not found or no change detected';  
    END IF;  
END //  
DELIMITER ;  

-- Stored Procedure: DeletePrescriptionDetail  
DELIMITER //  
CREATE PROCEDURE DeletePrescriptionDetail(  
    IN p_prescriptionID INT,  
    IN p_drugID INT  
)  
BEGIN  
    DELETE FROM PrescriptionDetail WHERE prescriptionID = p_prescriptionID AND drugID = p_drugID;  
    IF ROW_COUNT() = 0 THEN  
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'PrescriptionDetail entry not found';  
    END IF;  
END //  
DELIMITER ;








-- Create sample data for testing
-- Insert doctors
CALL AddDoctor('310925481237', 'Dr. Nikhil Rao', 'Cardiology', 15);
CALL AddDoctor('487230194852', 'Dr. Tanya Verma', 'Pediatrics', 8);
CALL AddDoctor('920384752930', 'Dr. Kiran Sethi', 'Orthopedics', 12);
CALL AddDoctor('175039284750', 'Dr. Aarti Bansal', 'Neurology', 20);
CALL AddDoctor('692830471923', 'Dr. Mohan Pillai', 'Dermatology', 10);

-- Insert patients
CALL AddPatient('720493812740', 'Rohit Das', 'Aurangabad', 35, '310925481237');
CALL AddPatient('918273645091', 'Divya Jain', 'Surat', 28, '487230194852');
CALL AddPatient('384756192837', 'Naveen Rao', 'Patna', 42, '920384752930');
CALL AddPatient('659182374501', 'Maya Kapoor', 'Nagpur', 25, '175039284750');
CALL AddPatient('204817365928', 'Ajay Sen', 'Raipur', 55, '692830471923');
CALL AddPatient('817263948120', 'Preeti Nanda', 'Bhopal', 30, '310925481237');
CALL AddPatient('635918273640', 'Varun Khurana', 'Vijayawada', 47, '487230194852');
CALL AddPatient('193847562304', 'Ritu Arora', 'Coimbatore', 32, '920384752930');
CALL AddPatient('748192036482', 'Manoj Tyagi', 'Kanpur', 50, '175039284750');
CALL AddPatient('857392047152', 'Isha Mehra', 'Guwahati', 29, '692830471923');

-- Insert pharmaceutical companies
CALL AddPharmaceuticalCompany('MediCore Labs', '9123456780');
CALL AddPharmaceuticalCompany('HealthCure Inc.', '9234567891');
CALL AddPharmaceuticalCompany('BioAid Pharma', '9345678912');
CALL AddPharmaceuticalCompany('WellnessMed', '9456789123');
CALL AddPharmaceuticalCompany('NutraPlus', '9567891234');

-- Insert pharmacies
CALL AddPharmacy('MediPoint', 'Sector 22, Indore', '9988776655');
CALL AddPharmacy('HealthMart', 'Green Park, Surat', '8877665544');
CALL AddPharmacy('CureWell', 'Hill Road, Patna', '7766554433');
CALL AddPharmacy('LifeMeds', 'Anna Nagar, Nagpur', '6655443322');
CALL AddPharmacy('PharmaOne', 'CBD, Raipur', '5544332211');

-- Insert drugs
CALL AddDrug('Healcin 500', 'C8H9NO2', 'MediCore Labs');
CALL AddDrug('Bactrinol 250', 'C16H19N3O5S', 'HealthCure Inc.');
CALL AddDrug('Glucomet 500', 'C4H11N5', 'BioAid Pharma');
CALL AddDrug('Cholstat 10', 'C33H35FN2O5', 'WellnessMed');
CALL AddDrug('Gastrozol 20', 'C17H19N3O3S', 'NutraPlus');
CALL AddDrug('Tranquix 5', 'C16H13ClN2O', 'MediCore Labs');
CALL AddDrug('Zefronil 1g', 'C18H18N8O7S3', 'HealthCure Inc.');
CALL AddDrug('Heartel 10', 'C21H31N3O5', 'BioAid Pharma');
CALL AddDrug('Lipocure 20', 'C25H38O5', 'WellnessMed');
CALL AddDrug('Mycinol 500', 'C38H72N2O12', 'NutraPlus');
CALL AddDrug('Painex 400', 'C13H18O2', 'MediCore Labs');
CALL AddDrug('Pressa 50', 'C22H23ClN6O', 'HealthCure Inc.');
CALL AddDrug('Cardinorm 5', 'C20H25ClN2O5', 'BioAid Pharma');
CALL AddDrug('Betocare 25', 'C15H25NO3', 'WellnessMed');
CALL AddDrug('Zetriz 10', 'C21H25ClN2O3', 'NutraPlus');

-- Add drugs to pharmacies with prices and stock
-- For MediPoint
CALL AddDrugToPharmacy(1, 1, 6.10, 100);
CALL AddDrugToPharmacy(1, 2, 13.20, 80);
CALL AddDrugToPharmacy(1, 3, 9.00, 120);
CALL AddDrugToPharmacy(1, 4, 23.45, 50);
CALL AddDrugToPharmacy(1, 5, 19.10, 75);
CALL AddDrugToPharmacy(1, 6, 14.75, 60);
CALL AddDrugToPharmacy(1, 7, 36.80, 40);
CALL AddDrugToPharmacy(1, 8, 21.30, 90);
CALL AddDrugToPharmacy(1, 9, 26.00, 65);
CALL AddDrugToPharmacy(1, 10, 44.50, 30);

-- For HealthMart
CALL AddDrugToPharmacy(2, 1, 6.35, 90);
CALL AddDrugToPharmacy(2, 2, 13.75, 70);
CALL AddDrugToPharmacy(2, 6, 16.00, 55);
CALL AddDrugToPharmacy(2, 11, 10.10, 110);
CALL AddDrugToPharmacy(2, 12, 29.10, 45);
CALL AddDrugToPharmacy(2, 13, 33.25, 60);
CALL AddDrugToPharmacy(2, 14, 20.00, 80);
CALL AddDrugToPharmacy(2, 15, 15.25, 95);
CALL AddDrugToPharmacy(2, 3, 9.10, 100);
CALL AddDrugToPharmacy(2, 4, 24.00, 40);

-- Create contracts between pharmacies and pharmaceutical companies
CALL AddContract(1, 'MediCore Labs', '2024-01-01', '2024-12-31', 'Pain and fever medications', 'Nidhi Sharma');
CALL AddContract(1, 'HealthCure Inc.', '2024-02-15', '2025-02-14', 'Antibiotics and cardiac drugs', 'Deepak Rao');
CALL AddContract(2, 'BioAid Pharma', '2024-03-10', '2025-03-09', 'Diabetes and BP control', 'Ramesh Varma');
CALL AddContract(2, 'WellnessMed', '2024-04-05', '2025-04-04', 'Cholesterol and heart meds', 'Juhi Joshi');
CALL AddContract(3, 'NutraPlus', '2024-05-20', '2025-05-19', 'Gastro and allergy meds', 'Kunal Khanna');

-- Create some prescriptions
CALL AddPrescription('720493812740', '310925481237', '2024-03-15');
SET @prescription1 = LAST_INSERT_ID();
CALL AddDrugToPrescription(@prescription1, 1, 20);
CALL AddDrugToPrescription(@prescription1, 4, 10);

CALL AddPrescription('918273645091', '487230194852', '2024-03-18');
SET @prescription2 = LAST_INSERT_ID();
CALL AddDrugToPrescription(@prescription2, 2, 15);
CALL AddDrugToPrescription(@prescription2, 15, 10);

CALL AddPrescription('384756192837', '920384752930', '2024-03-20');
SET @prescription3 = LAST_INSERT_ID();
CALL AddDrugToPrescription(@prescription3, 11, 30);
CALL AddDrugToPrescription(@prescription3, 3, 60);

CALL AddPrescription('659182374501', '175039284750', '2024-03-22');
SET @prescription4 = LAST_INSERT_ID();
CALL AddDrugToPrescription(@prescription4, 13, 30);
CALL AddDrugToPrescription(@prescription4, 14, 30);

CALL AddPrescription('204817365928', '692830471923', '2024-03-25');
SET @prescription5 = LAST_INSERT_ID();
CALL AddDrugToPrescription(@prescription5, 5, 30);

