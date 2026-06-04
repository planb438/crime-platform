# crime-platform

Crime database schema

---

Step 1: Create the schema
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "CREATE SCHEMA IF NOT EXISTS crime_data;"

---

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

---

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

---

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

---

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

---

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

---

Step 7: Insert sample incidents
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incidents (report_number, incident_date, district_id, crime_type_id, location_name, description) VALUES
('R2025001', '2025-01-15', 1, 1, 'Downtown Mall', 'Altercation between two individuals'),
('R2025002', '2025-01-16', 2, 2, 'Residential Area', 'Break and enter, electronics stolen'),
('R2025003', '2025-01-20', 1, 5, 'Gas Station', 'Armed robbery at convenience store');
"

---

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

---

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

Enhanced Database Schema for Real Crime Reports
---

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

---

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

---

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

---

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

---

Step 5: Create police units table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.police_units (
    unit_id SERIAL PRIMARY KEY,
    unit_name VARCHAR(100),
    unit_type VARCHAR(50)
);
"

---

Step 6: Create incident_units link table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.incident_units (
    incident_id INTEGER REFERENCES crime_data.incidents(incident_id),
    unit_id INTEGER REFERENCES crime_data.police_units(unit_id)
);
"

---

Step 7: Add new columns to incidents table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
ALTER TABLE crime_data.incidents 
ADD COLUMN IF NOT EXISTS incident_type VARCHAR(50),
ADD COLUMN IF NOT EXISTS technology_involved BOOLEAN DEFAULT FALSE,
ADD COLUMN IF NOT EXISTS shotspotter_detected BOOLEAN DEFAULT FALSE,
ADD COLUMN IF NOT EXISTS narrative TEXT;
"

---

Step 8: Insert police units
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.police_units (unit_name, unit_type) VALUES
('Mobile Division', 'Patrol'),
('Financial Crimes Investigation Branch', 'Investigative'),
('Police Bomb Squad', 'Specialized'),
('Anti-Gang and Firearms Investigation Task Force', 'Task Force');
"

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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


---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

Processed 3700 incidents...

Import complete!
   - 3718 incidents imported
   - 0 errors encountered
ubuntu@ip-10-0-1-220:~/Python-Scraper$
🎉 Excellent! Successfully imported 3,718 incidents from the RBDF crime reports! That's a substantial dataset for analysis.

---

Let's explore your imported crime data
---

1. Quick overview of the data
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    COUNT(*) as total_incidents,
    COUNT(DISTINCT incident_type) as crime_categories,
    MIN(incident_date) as earliest_report,
    MAX(incident_date) as latest_report,
    COUNT(DISTINCT location_name) as locations
FROM crime_data.incidents;
"

---

2. Crime type distribution
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    incident_type,
    COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER(), 1) as percentage
FROM crime_data.incidents
GROUP BY incident_type
ORDER BY count DESC
LIMIT 10;
"

---

3. Monthly crime trends (2026)
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    TO_CHAR(DATE_TRUNC('month', incident_date), 'YYYY-MM') as month,
    COUNT(*) as total_incidents,
    COUNT(CASE WHEN incident_type = 'Shooting' THEN 1 END) as shootings,
    COUNT(CASE WHEN incident_type = 'Firearm Related' THEN 1 END) as firearm_related
FROM crime_data.incidents
WHERE incident_date >= '2026-01-01'
GROUP BY DATE_TRUNC('month', incident_date)
ORDER BY month DESC;
"

---

4. Top locations for crime
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    location_name,
    COUNT(*) as incidents,
    COUNT(DISTINCT incident_type) as crime_types
FROM crime_data.incidents
WHERE location_name NOT IN ('Unknown', 'NEW PROVIDENCE')
GROUP BY location_name
ORDER BY incidents DESC
LIMIT 15;
"

---

5. Weapons and injuries analysis
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
SELECT 
    incident_type,
    SUM(injuries_count) as total_injuries,
    SUM(fatalities_count) as total_fatalities,
    COUNT(*) as total_incidents
FROM crime_data.incidents
WHERE injuries_count > 0 OR fatalities_count > 0
GROUP BY incident_type
ORDER BY total_fatalities DESC;
"

---

6. Create a summary view for your dashboard
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics << 'EOF'
CREATE OR REPLACE VIEW crime_data.crime_summary_stats AS
SELECT 
    DATE(incident_date) as date,
    incident_type,
    location_name,
    COUNT(*) as incident_count,
    SUM(injuries_count) as injuries,
    SUM(fatalities_count) as fatalities
FROM crime_data.incidents
WHERE incident_date IS NOT NULL
GROUP BY DATE(incident_date), incident_type, location_name
ORDER BY date DESC;
EOF

---

7. Check the summary view
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT * FROM crime_data.crime_summary_stats LIMIT 20;"
What's Next?
Your database now has 3,718 real crime incidents from the Royal Bahamas Police Force. You can now:

Option 1: Deploy Grafana Dashboard
Visualize crime trends, heatmaps, and time-series analysis

Option 2: Deploy pgAdmin
Manage and query your data with a GUI

Option 3: Create API Layer
Build a REST API for mobile apps or internal tools

Option 4: Run Advanced Analytics
Crime prediction models

Hotspot analysis

Officer safety alerts

Which would you like to do next? I recommend Grafana first so you can see your 3,718 incidents visualized on dashboards!

---

Improvement for Manual Entry:
---

Step 1: Add helper columns for manual entry
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "ALTER TABLE crime_data.incidents ADD COLUMN IF NOT EXISTS reporter_name VARCHAR(100);"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "ALTER TABLE crime_data.incidents ADD COLUMN IF NOT EXISTS reporter_badge VARCHAR(20);"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "ALTER TABLE crime_data.incidents ADD COLUMN IF NOT EXISTS entered_by VARCHAR(50) DEFAULT 'manual_entry';"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "ALTER TABLE crime_data.incidents ADD COLUMN IF NOT EXISTS last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP;"

---

Step 2: Create view for manual entry
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE OR REPLACE VIEW crime_data.manual_entry_form AS
SELECT 
    incident_id,
    report_number,
    incident_date,
    incident_type,
    location_name,
    description,
    injuries_count,
    fatalities_count,
    reporter_name,
    reporter_badge,
    status
FROM crime_data.incidents
WHERE entered_by = 'manual_entry'
ORDER BY incident_id DESC;
"

---

Step 3: Create function to auto-generate report numbers
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE OR REPLACE FUNCTION generate_report_number()
RETURNS TRIGGER AS \$\$
BEGIN
    IF NEW.report_number IS NULL THEN
        NEW.report_number := 'MANUAL-' || TO_CHAR(NOW(), 'YYYYMMDD') || '-' || LPAD(NEW.incident_id::text, 4, '0');
    END IF;
    RETURN NEW;
END;
\$\$ LANGUAGE plpgsql;
"

---

Step 4: Create trigger for auto report numbers
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "DROP TRIGGER IF EXISTS auto_report_number ON crime_data.incidents;"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TRIGGER auto_report_number
    BEFORE INSERT ON crime_data.incidents
    FOR EACH ROW
    EXECUTE FUNCTION generate_report_number();
"

---

Step 5: Add validation constraints
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
ALTER TABLE crime_data.incidents 
ADD CONSTRAINT valid_incident_date 
CHECK (incident_date <= CURRENT_DATE);
"

---

Step 6: Create dropdown helper table
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.incident_types_lookup (
    type_id SERIAL PRIMARY KEY,
    type_name VARCHAR(50) UNIQUE NOT NULL,
    category VARCHAR(50),
    is_violent BOOLEAN DEFAULT FALSE
);
"

---

Step 7: Insert common types
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Assault', 'Violent Crime', true)
ON CONFLICT (type_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Burglary', 'Property Crime', false)
ON CONFLICT (type_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Theft', 'Property Crime', false)
ON CONFLICT (type_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Homicide', 'Violent Crime', true)
ON CONFLICT (type_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Robbery', 'Violent Crime', true)
ON CONFLICT (type_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Shooting', 'Weapons Violation', true)
ON CONFLICT (type_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Drug Related', 'Narcotics', false)
ON CONFLICT (type_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Sexual Assault', 'Violent Crime', true)
ON CONFLICT (type_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Stolen Vehicle', 'Property Crime', false)
ON CONFLICT (type_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incident_types_lookup (type_name, category, is_violent) VALUES
('Firearm Possession', 'Weapons Violation', true)
ON CONFLICT (type_name) DO NOTHING;
"

---

Step 8: Create lookup for status values
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
CREATE TABLE IF NOT EXISTS crime_data.status_lookup (
    status_id SERIAL PRIMARY KEY,
    status_name VARCHAR(30) UNIQUE NOT NULL
);
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.status_lookup (status_name) VALUES ('under_investigation')
ON CONFLICT (status_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.status_lookup (status_name) VALUES ('closed')
ON CONFLICT (status_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.status_lookup (status_name) VALUES ('unsolved')
ON CONFLICT (status_name) DO NOTHING;
"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.status_lookup (status_name) VALUES ('referred_to_court')
ON CONFLICT (status_name) DO NOTHING;
"

---

Step 9: Verify everything worked
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT * FROM crime_data.incident_types_lookup;"
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT * FROM crime_data.status_lookup;"
Step 10: Test a manual entry
bash
kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
INSERT INTO crime_data.incidents (
    incident_date,
    incident_type,
    location_name,
    description,
    reporter_name,
    reporter_badge
) VALUES (
    '2026-06-04',
    'Assault',
    'Downtown Nassau',
    'Officer responded to disturbance at bar',
    'Officer Johnson',
    'B1234'
)
RETURNING report_number, incident_id;
"

---

Summary: Your Database Structure
Aspect	Rating	Notes
Manual Entry Ready	⭐⭐⭐⭐	Good, with minor improvements added
Validation	⭐⭐⭐	Has date validation, could add more
Auto-generation	⭐⭐⭐⭐	Auto report numbers and timestamps
Lookup tables	⭐⭐⭐⭐⭐	Added dropdown helpers
Audit trail	⭐⭐⭐	Tracks who entered data
Verdict: Your database is well-structured for manual entry now that we've added the helper tables and auto-generation features. You can enter data with just 3-4 fields, and the rest auto-populates.

