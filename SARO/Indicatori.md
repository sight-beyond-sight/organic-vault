1) Total possible users: Fk Marketing (ASE): 2600
2) Opt-in users (database): /home/dev/conn/ase-vpn/ASE.db (172 rn)
Every week, re-run script on this db. Input is consent form.

e.g.
/home/dev/conn/ase-vpn/saro_consent_13_05.csv
- Run /home/dev/conn/ase-vpn/add_students.py (has a field with path to csv input): this adds these students to the ASE db filter
- Run /home/dev/conn/ase-vpn/scrape_db2.py This rebuilds our local database
- Then you can re-check student number

3) How many users answer SARO?
	1) AdminPanel User Analysis (press Show All)
	2) Keep in mind they dont automatically get a message from SARO when added  to DB
	3) Divide total interactions by number of users who answer to get average/median  answers
	4) Need last user message script

4) Flow table
	1) Need flow categorizations script (look through all chats, count them)
	2) Reply rate per flow
	3) Avg conv rate per flow