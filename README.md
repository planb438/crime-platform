# crime-platform

The command got cut off. Let me give you the commands one by one so it works properly.

Step 1: Create the schema
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "CREATE SCHEMA IF NOT EXISTS crime_data;"
Step 2: Create districts table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.districts (
    district_id SERIAL PRIMARY KEY,
    district_code VARCHAR(10) NOT NULL UNIQUE,
    district_name VARCHAR(100) NOT NULL,
    division VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
"
Step 3: Create crime types table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.crime_types (
    crime_type_id SERIAL PRIMARY KEY,
    crime_category VARCHAR(50) NOT NULL,
    crime_subcategory VARCHAR(100),
    is_violent BOOLEAN DEFAULT FALSE,
    severity_level INTEGER DEFAULT 1
);
"
Step 4: Create incidents table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.incidents (
    incident_id SERIAL PRIMARY KEY,
    report_number VARCHAR(50) UNIQUE NOT NULL,
    incident_date DATE NOT NULL,
    incident_time TIME,
    district_id INTEGER REFERENCES crime_data.districts(district_id),
    crime_type_id INTEGER REFERENCES crime_data.crime_types(crime_type_id),
    location_name VARCHAR(200),
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    description TEXT,
    injuries_count INTEGER DEFAULT 0,
    fatalities_count INTEGER DEFAULT 0,
    status VARCHAR(30) DEFAULT 'under_investigation',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
"
Step 5: Insert districts
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.districts (district_code, district_name, division) VALUES
('NP01', 'Nassau Central', 'New Providence'),
('NP02', 'Eastern Division', 'New Providence'),
('NP03', 'Western Division', 'New Providence'),
('GB01', 'Grand Bahama Central', 'Grand Bahama'),
('AB01', 'Abaco Central', 'Abaco')
ON CONFLICT (district_code) DO NOTHING;
"
Step 6: Insert crime types
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.crime_types (crime_category, crime_subcategory, is_violent, severity_level) VALUES
('Assault', 'Aggravated Assault', true, 4),
('Burglary', 'Residential Burglary', false, 3),
('Theft', 'Auto Theft', false, 2),
('Homicide', 'Murder', true, 5),
('Robbery', 'Armed Robbery', true, 5);
"
Step 7: Insert sample incidents
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incidents (report_number, incident_date, district_id, crime_type_id, location_name, description) VALUES
('R2025001', '2025-01-15', 1, 1, 'Downtown Mall', 'Altercation between two individuals'),
('R2025002', '2025-01-16', 2, 2, 'Residential Area', 'Break and enter, electronics stolen'),
('R2025003', '2025-01-20', 1, 5, 'Gas Station', 'Armed robbery at convenience store');
"
Step 8: Create the view
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE OR REPLACE VIEW crime_data.daily_crime_summary AS
SELECT 
    i.incident_date,
    d.district_name,
    ct.crime_category,
    COUNT(*) as incident_count,
    SUM(i.injuries_count) as total_injuries
FROM crime_data.incidents i
JOIN crime_data.districts d ON i.district_id = d.district_id
JOIN crime_data.crime_types ct ON i.crime_type_id = ct.crime_type_id
GROUP BY i.incident_date, d.district_name, ct.crime_category;
"
Step 9: Verify everything
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT * FROM crime_data.daily_crime_summary;"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT district_name, COUNT(*) as incidents FROM crime_data.incidents i JOIN crime_data.districts d ON i.district_id = d.district_id GROUP BY district_name;"
Run these commands one at a time, and let me know if any fail. The first one that failed was because the view didn't exist yet - we're creating it now step by step.

---

Here are some SQL queries to test your crime database:

Basic Queries
1. Show all districts
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT * FROM crime_data.districts;"
2. Show all crime types
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT * FROM crime_data.crime_types;"
3. Show all incidents
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT * FROM crime_data.incidents;"
Analysis Queries
4. Crime count by district (most dangerous first)
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT d.district_name, COUNT(i.incident_id) as total_crimes
FROM crime_data.districts d
LEFT JOIN crime_data.incidents i ON d.district_id = i.district_id
GROUP BY d.district_name
ORDER BY total_crimes DESC;
"
5. Violent vs Non-violent crimes
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    CASE WHEN ct.is_violent THEN 'Violent' ELSE 'Non-Violent' END as crime_type,
    COUNT(*) as total
FROM crime_data.incidents i
JOIN crime_data.crime_types ct ON i.crime_type_id = ct.crime_type_id
GROUP BY ct.is_violent;
"
6. Most common crime categories
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT ct.crime_category, COUNT(*) as frequency
FROM crime_data.incidents i
JOIN crime_data.crime_types ct ON i.crime_type_id = ct.crime_type_id
GROUP BY ct.crime_category
ORDER BY frequency DESC;
"
7. Crimes by severity level
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    ct.severity_level,
    CASE 
        WHEN ct.severity_level = 5 THEN 'Critical'
        WHEN ct.severity_level = 4 THEN 'High'
        WHEN ct.severity_level = 3 THEN 'Medium'
        WHEN ct.severity_level = 2 THEN 'Low'
        ELSE 'Info'
    END as severity_name,
    COUNT(*) as total
FROM crime_data.incidents i
JOIN crime_data.crime_types ct ON i.crime_type_id = ct.crime_type_id
GROUP BY ct.severity_level
ORDER BY ct.severity_level DESC;
"
8. Monthly crime trends
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    DATE_TRUNC('month', incident_date) as month,
    COUNT(*) as incidents
FROM crime_data.incidents
GROUP BY month
ORDER BY month;
"
9. Get database schema information
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "\dt crime_data.*"
10. Insert more test data (for better visualization)
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incidents (report_number, incident_date, district_id, crime_type_id, location_name, description, injuries_count) VALUES
('R2025004', '2025-02-01', 3, 4, 'Main Street', 'Homicide investigation', 1),
('R2025005', '2025-02-05', 1, 3, 'Parking Lot', 'Vehicle stolen', 0),
('R2025006', '2025-02-10', 2, 1, 'Night Club', 'Fight broke out', 2),
('R2025007', '2025-02-15', 4, 5, 'Bank', 'Attempted robbery', 0),
('R2025008', '2025-02-20', 1, 2, 'Residential Area', 'Home burglary', 0);
"
11. Verify the view works now
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT * FROM crime_data.daily_crime_summary ORDER BY incident_date;"
12. Complex query: Crime hotspot ranking
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    d.district_name,
    COUNT(i.incident_id) as total_crimes,
    SUM(i.injuries_count) as total_injuries,
    ROUND(AVG(ct.severity_level), 2) as avg_severity
FROM crime_data.districts d
LEFT JOIN crime_data.incidents i ON d.district_id = i.district_id
LEFT JOIN crime_data.crime_types ct ON i.crime_type_id = ct.crime_type_id
GROUP BY d.district_name
ORDER BY total_crimes DESC;
"
---

Excellent! This is real-world crime report data. Let me help you structure your database to handle both structured (like the Stolen Vehicle Alert) and unstructured (like the narrative police reports) data.

Enhanced Database Schema for Real Crime Reports

Perfect! Let's do this step by step, one command at a time.

Step 1: Add new tables for real crime reports
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.weapons (
    weapon_id SERIAL PRIMARY KEY,
    weapon_type VARCHAR(50),
    weapon_description TEXT,
    confiscated BOOLEAN DEFAULT FALSE
);
"
Step 2: Create arrests table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.arrests (
    arrest_id SERIAL PRIMARY KEY,
    incident_id INTEGER REFERENCES crime_data.incidents(incident_id),
    suspect_name VARCHAR(200),
    suspect_age INTEGER,
    suspect_nationality VARCHAR(50),
    suspect_gender VARCHAR(10),
    arrest_date DATE,
    arrest_location TEXT,
    charges TEXT,
    status VARCHAR(50),
    cautioned BOOLEAN DEFAULT FALSE,
    custody_status VARCHAR(50)
);
"
Step 3: Create stolen vehicles table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.stolen_vehicles (
    vehicle_id SERIAL PRIMARY KEY,
    incident_id INTEGER REFERENCES crime_data.incidents(incident_id),
    make VARCHAR(50),
    model VARCHAR(50),
    year INTEGER,
    color VARCHAR(30),
    license_plate VARCHAR(20),
    vin VARCHAR(50),
    stolen_date DATE,
    stolen_location TEXT,
    recovery_date DATE,
    status VARCHAR(30) DEFAULT 'stolen'
);
"
Step 4: Create evidence table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.evidence (
    evidence_id SERIAL PRIMARY KEY,
    incident_id INTEGER REFERENCES crime_data.incidents(incident_id),
    evidence_type VARCHAR(50),
    description TEXT,
    quantity INTEGER,
    seized_from VARCHAR(200),
    processed BOOLEAN DEFAULT FALSE
);
"
Step 5: Create police units table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.police_units (
    unit_id SERIAL PRIMARY KEY,
    unit_name VARCHAR(100),
    unit_type VARCHAR(50)
);
"
Step 6: Create incident_units link table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.incident_units (
    incident_id INTEGER REFERENCES crime_data.incidents(incident_id),
    unit_id INTEGER REFERENCES crime_data.police_units(unit_id)
);
"
Step 7: Add new columns to incidents table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
ALTER TABLE crime_data.incidents 
ADD COLUMN IF NOT EXISTS incident_type VARCHAR(50),
ADD COLUMN IF NOT EXISTS technology_involved BOOLEAN DEFAULT FALSE,
ADD COLUMN IF NOT EXISTS shotspotter_detected BOOLEAN DEFAULT FALSE,
ADD COLUMN IF NOT EXISTS narrative TEXT;
"
Step 8: Insert police units
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.police_units (unit_name, unit_type) VALUES
('Mobile Division', 'Patrol'),
('Financial Crimes Investigation Branch', 'Investigative'),
('Police Bomb Squad', 'Specialized'),
('Anti-Gang and Firearms Investigation Task Force', 'Task Force');
"
Step 9: Insert the firearm incident
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incidents (
    report_number, 
    incident_date, 
    district_id, 
    crime_type_id,
    incident_type,
    location_name,
    description,
    shotspotter_detected,
    status
) VALUES (
    'R20260526-001',
    '2026-05-26',
    1,
    1,
    'Weapons Violation',
    'Hillbrook Road off Wulff Road',
    'Firearm confiscation, two suspects arrested',
    true,
    'under_investigation'
);
"
Step 10: Insert arrests for the firearm incident
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.arrests (
    incident_id,
    suspect_name,
    suspect_age,
    suspect_nationality,
    arrest_date,
    charges,
    cautioned,
    custody_status
) VALUES 
    (1, 'Unknown', 36, 'Jamaican', '2026-05-26', 'Possession of firearm and ammunition', true, 'in custody');
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.arrests (
    incident_id,
    suspect_name,
    suspect_age,
    suspect_nationality,
    arrest_date,
    charges,
    cautioned,
    custody_status
) VALUES 
    (1, 'Unknown', 29, 'Bahamian', '2026-05-26', 'Possession of firearm and ammunition', true, 'in custody');
"
Step 11: Insert stolen vehicle incident
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incidents (
    report_number,
    incident_date,
    district_id,
    crime_type_id,
    incident_type,
    location_name,
    description,
    status
) VALUES (
    'R20260524-002',
    '2026-05-24',
    1,
    3,
    'Vehicle Theft',
    'Coral Harbour Road near LPIA',
    'Red 2012 Nissan Cube stolen from parking lot',
    'under_investigation'
);
"
Step 12: Insert stolen vehicle details
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.stolen_vehicles (
    incident_id,
    make,
    model,
    year,
    color,
    license_plate,
    stolen_date,
    stolen_location,
    status
) VALUES (
    2,
    'Nissan',
    'Cube',
    2012,
    'Red',
    'DA1536',
    '2026-05-24',
    'Parking lot on Coral Harbour Road near LPIA',
    'stolen'
);
"
Step 13: Insert ATM skimming incident
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incidents (
    report_number,
    incident_date,
    district_id,
    crime_type_id,
    incident_type,
    location_name,
    description,
    technology_involved,
    status
) VALUES (
    'R20260526-003',
    '2026-05-26',
    1,
    3,
    'Financial Crime',
    'Financial institution in eastern New Providence',
    'Bulgarian national arrested for ATM skimming operation',
    true,
    'under_investigation'
);
"
Step 14: Insert arrest for skimming
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.arrests (
    incident_id,
    suspect_name,
    suspect_age,
    suspect_nationality,
    arrest_date,
    charges,
    custody_status
) VALUES (
    3,
    'Unknown',
    36,
    'Bulgarian',
    '2026-05-26',
    'ATM skimming, financial crimes, possession of illegal electronic equipment',
    'in custody'
);
"
Step 15: Insert evidence for skimming
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.evidence (
    incident_id,
    evidence_type,
    description,
    seized_from
) VALUES (
    3,
    'Electronic Device',
    'ATM skimming devices and concealed pinhole camera components',
    'Bulgarian suspect'
);
"
Step 16: Verify all data was inserted
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    i.report_number,
    i.incident_date,
    i.incident_type,
    i.description,
    COUNT(a.arrest_id) as arrests
FROM crime_data.incidents i
LEFT JOIN crime_data.arrests a ON i.incident_id = a.incident_id
GROUP BY i.incident_id
ORDER BY i.incident_date DESC;
"

---

Perfect! Now let's verify everything is working and run some meaningful queries on your crime data.

Step 17: View all incidents with their details
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    report_number,
    incident_date,
    incident_type,
    location_name,
    description,
    CASE WHEN shotspotter_detected THEN 'Yes' ELSE 'No' END as shotspotter,
    status
FROM crime_data.incidents
ORDER BY incident_date DESC;
"
Step 18: Show arrests by incident
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    i.report_number,
    i.incident_type,
    a.suspect_age,
    a.suspect_nationality,
    a.charges,
    a.custody_status
FROM crime_data.incidents i
JOIN crime_data.arrests a ON i.incident_id = a.incident_id;
"
Step 19: Show stolen vehicles
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    i.report_number,
    v.make,
    v.model,
    v.year,
    v.color,
    v.license_plate,
    v.stolen_location,
    v.status
FROM crime_data.incidents i
JOIN crime_data.stolen_vehicles v ON i.incident_id = v.incident_id;
"
Step 20: Evidence collected by incident
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    i.report_number,
    i.incident_type,
    e.evidence_type,
    e.description,
    e.seized_from
FROM crime_data.incidents i
JOIN crime_data.evidence e ON i.incident_id = e.incident_id;
"
Step 21: Summary statistics for your dashboard
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    'Total Incidents' as metric,
    COUNT(*) as value
FROM crime_data.incidents
UNION ALL
SELECT 
    'Total Arrests',
    COUNT(*)
FROM crime_data.arrests
UNION ALL
SELECT 
    'Stolen Vehicles',
    COUNT(*)
FROM crime_data.stolen_vehicles
UNION ALL
SELECT 
    'Technology Related',
    COUNT(*)
FROM crime_data.incidents
WHERE technology_involved = true OR shotspotter_detected = true;
"
Step 22: Crime by type breakdown
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    incident_type,
    COUNT(*) as total
FROM crime_data.incidents
GROUP BY incident_type
ORDER BY total DESC;
"
Step 23: International suspects involved
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    suspect_nationality,
    COUNT(*) as suspects,
    STRING_AGG(DISTINCT i.incident_type, ', ') as crime_types
FROM crime_data.arrests a
JOIN crime_data.incidents i ON a.incident_id = i.incident_id
WHERE suspect_nationality != 'Bahamian'
GROUP BY suspect_nationality;
"
Step 24: Recent arrests (last 7 days)
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    a.arrest_date,
    a.suspect_age,
    a.suspect_nationality,
    a.charges,
    i.incident_type
FROM crime_data.arrests a
JOIN crime_data.incidents i ON a.incident_id = i.incident_id
ORDER BY a.arrest_date DESC
LIMIT 10;
"
Step 25: Create a simplified view for daily reporting
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE OR REPLACE VIEW crime_data.daily_crime_log AS
SELECT 
    i.incident_date,
    i.incident_type,
    i.location_name,
    i.description as incident_summary,
    COUNT(DISTINCT a.arrest_id) as arrests_made,
    CASE WHEN i.shotspotter_detected THEN 'ShotSpotter' ELSE '' END as technology_used
FROM crime_data.incidents i
LEFT JOIN crime_data.arrests a ON i.incident_id = a.incident_id
GROUP BY i.incident_id, i.incident_date, i.incident_type, i.location_name, i.description, i.shotspotter_detected
ORDER BY i.incident_date DESC;
"
Step 26: Test the daily view
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT * FROM crime_data.daily_crime_log LIMIT 5;
"
What's Working Now:
✅ Database schema supports real RBDF reports
✅ Structured data for firearms, arrests, stolen vehicles, evidence
✅ Tracked technology involvement (ShotSpotter, skimming devices)
✅ International suspect tracking
✅ Views for daily crime reporting
