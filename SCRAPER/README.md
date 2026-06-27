
#### Step 1: Make incident_date nullable temporarily (or handle missing dates)
#### bash
    kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
    ALTER TABLE crime_data.incidents 
    ALTER COLUMN incident_date DROP NOT NULL;
#### "
#### Step 2: Use this corrected script that handles missing dates
#### bash
    cat > import_json_final.py << 'EOF'
    import json
    import psycopg2
    from datetime import datetime

    conn = psycopg2.connect(
        host="localhost",
        port=5432,
        database="crime_analytics",
        user="crime_analyst",
        password="RBDFcrime2025"
    )
    cursor = conn.cursor()

    with open('/tmp/rbdf_crime_reports.json', 'r') as f:
        reports = json.load(f)

    incident_count = 0
    error_count = 0

    for report in reports:
        # Handle report date - if missing, use a default or skip
        report_date = report.get('report_date')
        if report_date and isinstance(report_date, str):
            try:
                report_date = datetime.strptime(report_date, '%Y-%m-%d')
            except:
            report_date = None
        elif not report_date:
            report_date = None
    
        for incident in report.get('incidents', []):
            try:
                # Convert all values safely
                narrative_text = str(incident.get('full_text', '') or '')[:2000]
                description_text = str(incident.get('specific_location', '') or '')[:500]
                crime_type = str(incident.get('crime_type', 'Unknown') or 'Unknown')[:100]
                location = str(incident.get('location', 'Unknown') or 'Unknown')[:100]
            
                # Handle boolean values
                injuries = incident.get('injuries', False)
                if injuries is None:
                    injuries = False
            
                fatalities = incident.get('fatalities', False)
                if fatalities is None:
                    fatalities = False
            
               cursor.execute("""
                    INSERT INTO crime_data.incidents (
                        report_number,
                        incident_date,
                        incident_type,
                        location_name,
                        description,
                        narrative,
                        injuries_count,
                        fatalities_count,
                        status,
                        created_at
                    ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                    ON CONFLICT (report_number) DO NOTHING
                """, (
                    f"R{report['report_id']}",
                    report_date,
                    crime_type,
                    location,
                    description_text,
                    narrative_text,
                    1 if injuries else 0,
                    1 if fatalities else 0,
                    'scraped',
                    datetime.now()
                ))
                incident_count += 1
            
                if incident_count % 50 == 0:
                    print(f"Processed {incident_count} incidents...")
                
            except Exception as e:
                error_count += 1
                print(f"Error with report {report.get('report_id', 'unknown')}: {e}")
    
    conn.commit()
    print(f"\nImport complete!")
    print(f"   - {incident_count} incidents imported")
    print(f"   - {error_count} errors encountered")

    cursor.close()
    conn.close()
    EOF

#### Step 3: Copy and run the final script
#### bash
    kubectl cp import_json_final.py postgres/postgres-8b87874bc-5lqrw:/tmp/import_json_final.py
#### bash
    kubectl exec -n postgres postgres-8b87874bc-5lqrw -- python3 /tmp/import_json_final.py
#### Step 4: Verify the data
#### bash
# Check total count
    kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT COUNT(*) as total FROM crime_data.incidents;"
#### bash
# See what data was imported
    kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT report_number, incident_date, incident_type, location_name FROM crime_data.incidents LIMIT 10;"
#### bash
# Check for null dates
    kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "SELECT COUNT(*) as null_dates FROM crime_data.incidents WHERE incident_date IS NULL;"
#### Step 5: Fix null dates if needed (optional)
#### If you want to set a default date for null values:

#### bash
    kubectl exec -n postgres postgres-8b87874bc-5lqrw -- psql -U crime_analyst -d crime_analytics -c "
    UPDATE crime_data.incidents 
    SET incident_date = '2026-01-01' 
    WHERE incident_date IS NULL;
#### "
