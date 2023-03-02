# praca- podyplomowa 
import sqlite3
import xml.etree.ElementTree as ET
import os
from datetime import datetime
#tworzenie bazy danych w sqlite
conn = None
try:
    conn = sqlite3.connect("basedb.db")
    print(sqlite3.version)
    cursor = conn.cursor()
    sqlite_create_table_query = '''CREATE TABLE entries (    lead_sponsor         TEXT,
    collaborators_list   TEXT,
    intervention_type    TEXT,
    overall_status       TEXT,
    start_date           TEXT,
    completion_date      TEXT,
    phase                TEXT,
    study_type           TEXT,
    id
        primary key,
    detailed_description TEXT,
    months_spent         INTEGER,
    condition            TEXT);'''
    cursor.execute(sqlite_create_table_query)
except Error as e:
        print(e)
finally:
    if conn:
        conn.close()
        # ta funkcja/metoda usuwa ewentualne wystąpienia DNIA w miesiącu w danych 
def transform(s):
    string = s
    try :
        rem_start = s.index(" ")
        rem_end = s.index(",")+1
        string = s[0:rem_start:] + s[rem_end: : ]
    except:
        pass
    return string

# funkcja, która oblicza ile miesięcy upłynęło od rozpoczęcia do zakończenia;

def calculate_time_delta(start, finish):
    s = start
    s = transform(s)
    s = "1 " + s
    f = finish
    f = transform(f)
    f = "1 " + f
    sd = datetime.strptime(s, "%d %B %Y")
    fd = datetime.strptime(f, "%d %B %Y")
    months = (fd.year - sd.year) * 12 + (fd.month - sd.month)
    return months


# KLASA - obiekt zrobiony na potrzeby tego projektu. Każdej linijce kodu odpowiada jedno Entry


class Entry:
  def __init__(self, nct_id, lead_sponsor, collaborators_list, intervention_type, overall_status,
               start_date, completion_date, phase, study_type, lead_sponsor_type, detailed_description, condition):
    self.nct_id = nct_id
    self.lead_sponsor = lead_sponsor
    self.collaborators_list = collaborators_list
    self.intervention_type = intervention_type
    self.overall_status = overall_status
    self.start_date = start_date
    self.completion_date = completion_date
    self.phase = phase
    self.study_type = study_type
    self.lead_sponsor_type = lead_sponsor_type
    self.detailed_description = detailed_description
    self.condition = condition
    if (self.completion_date and self.start_date):
        self.months_spent = calculate_time_delta(self.start_date, self.completion_date)

#funkcja generująca kawałek tekstu, który jest potrzebny do przejścia po wszystkich folderach z tego archiwum,
# z którego korzystamy
def generate_folder_no(dir):
    if dir == 0:
        return "0000"
    if dir < 10 :
        return "000" + str(dir)
    if dir < 100 :
        return "00" + str(dir)
    if dir < 1000 :
        return "0" + str(dir)

# funkcja wykonująca operację na naszej bazie danych. przyjmuje obiekt klasy entry (którą zdefiniowaliśmy wyżej)
# wrzucamy wszystko w bazę sql

def insert_entry(entry, conn) :
    print("inserting the following data")
    sql = '''INSERT INTO entries(lead_sponsor,collaborators_list,intervention_type,overall_status,start_date,completion_date,phase, study_type, id, detailed_description, months_spent, condition)VALUES(
    ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)'''

    print("Sponsor lead:" + entry.lead_sponsor, "Other sponsors:" + entry.collaborators_list, entry.intervention_type, entry.overall_status, entry.start_date, entry.completion_date, entry.phase, entry.study_type, entry.nct_id, entry.detailed_description, entry.months_spent, entry.condition)

    cur = conn.cursor()
    cur.execute(sql,   (entry.lead_sponsor, entry.collaborators_list, entry.intervention_type, entry.overall_status, entry.start_date, entry.completion_date, entry.phase, entry.study_type, entry.nct_id, entry.detailed_description, entry.months_spent, entry.condition))
    # do tego momentu, tutaj dane się jeszcze nie wysłały - siedzą u nas w pamięci. Dlatego trzeba jeszcze 
    # wykonać commit - czyli zapisać dane - od tego momentu dane będą w tabeli
    conn.commit()

# funkcja przyjmująca ścieżkę na dysku, gdzie znajdują się pliki i obiekt typu database connection - czyli c
# cała funkcja jest zapętlona - powtarzamy to samo, aż przejdziemy po wszystkich plikach i folderach
def zrob_mie(path, c) :
    #robimy pętlę for, która przechodzi po wszystkich plikach w ścieżce
    for filename in os.listdir(path):
        #jeśli plik nie ma rozszerzenia .xml to go pomijamy
        if not filename.endswith('.xml'): continue
        #przy pomocy biblioteki os łączymy ścieżkę do pliku i nazwę pliku,
        fullname = os.path.join(path, filename)
        #po czym podajemy to do funkcji parse z biblioteki ET 
        tree = ET.parse(fullname)
        #ta biblioteka buduje nam mapę pliku xml - tak żebyśmy mogli się po niej poruszać.
        root = tree.getroot()
        item_lead_sponsor = ""
        item_lead_sponsor_type = ""
        item_overall_status = ""
        item_start_date = ""
        item_completion_date = ""
        item_study_type = ""
        item_nct_id = ""
        item_intervention_string = ""
        item_collaborators_string = ""
        item_detailed_description = ""
        item_condition = ""

        #najpierw sprawdzamy co jest zapisane w pliku pod adresem phase

        for item in root.findall("phase") :
            phase = item.text

            if((phase == "Phase 1") or (phase == "Phase 2") or (phase == "Phase 3") or (phase == "Phase 4") or
                    (" 1" in phase) or (" 2" in phase) or (" 3" in phase) or (" 4" in phase)):
                #jeśli faza nam odpowiada, to zapisujemy ją od razu do zmiennej, żebyśmy mogli sobie ją później zapisać
                item_phase = phase

                #dalej mamy pętle for, które wyszukują nam wystąpień elementów w nawiasach
                for sponsor in root.findall("sponsors/lead_sponsor/agency"):
                    item_lead_sponsor = sponsor.text

                for sponsor_type in root.findall("sponsors/lead_sponsor/agency_class"):
                    item_lead_sponsor_type = sponsor_type.text

                for collaborator in root.findall("sponsors/collaborator") :
                    agency_name_list = collaborator.findall("agency")
                    for agency in agency_name_list:
                        agency_name = agency.text

                    agency_type_list = collaborator.findall("agency_class")
                    for agency in agency_type_list:
                        agency_class = agency.text

                    item_collaborators_string += agency_name + ", " + agency_class + "; "

                for item in root.findall("intervention"):
                    intervention_a = item.findall("intervention_type")
                    for thisint in intervention_a:
                        intervention_type = thisint.text

                    intervention_b = item.findall("intervention_name")
                    for thisint in intervention_b:
                        intervention_name = thisint.text
                    item_intervention_string += (intervention_type + ": " + intervention_name + "; ")

                for item in root.findall("overall_status") :
                    item_overall_status = item.text
                for item in root.findall("start_date") :
                    item_start_date = item.text
                for item in root.findall("completion_date") :
                    item_completion_date = item.text
                for item in root.findall("study_type") :
                    item_study_type = item.text
                for item in root.findall("id_info/nct_id") :
                    item_nct_id = item.text
                for item in root.findall("detailed_description/textblock") :
                    item_detailed_description = item.text
                for item in root.findall("condition") :
                    item_condition = item_condition + "Condition: " + item.text + "; "
                    
                #tworzymy obiekt naszej klasy 
                # definiowaliśmy na początku; poniżej mówimy - robię nowy obiekt klasy Entry z parametrami jak wyżej
            
                new_entry = Entry(item_nct_id,item_lead_sponsor,item_collaborators_string,item_intervention_string,
                                  item_overall_status,item_start_date,item_completion_date, item_phase,item_study_type,
                                  item_lead_sponsor_type, item_detailed_description, item_condition)
                #jpo utworzeniu obiektu sprawdzamy czy on nam jest w ogóle potrzebny - tj. sprawdzamy
                # czy ma status Completed, czy ma podany czas startu projektu i czas zakończenia projektu. Jeśli tak to
                if(new_entry.overall_status == "Completed" and new_entry.completion_date and new_entry.start_date):
                    #wrzucamy go do naszej bazy danych (pierwszy argument funkcji to nasz obiekt 

                    insert_entry(new_entry, c)
if __name__ == '__main__':
    print('Running')
    #tworzymy połączenie z plikiem basedb.db później, będzie wykorzystany do analizy danych 
    conn = sqlite3.connect('basedb.db')
    for directory in range(0,572) :
        temp_directory = "C:/praca podyplomowa/NCT" + generate_folder_no(directory) + "xxxx"
        zrob_mie(temp_directory, conn)

    conn.close()
import pandas as pd
import numpy as np
import pyodbc
import sqlite3



con = sqlite3.connect("base.db")
df = pd.read_sql_query("SELECT * from entries", con)
print(df.head())
import sqlite3
import xml.etree.ElementTree as ET
import os
from datetime import datetime
import pandas as pd
import sqlalchemy
import seaborn as sns
import matplotlib.pyplot as plt
%matplotlib inline
import numpy as np
from matplotlib import ticker
sns.set()
sns.set_style("whitegrid")
#dodatkowo, odfiltrowujemy ostrzeżenia podnoszone przez bibliotekę pandas, są nam w tym momencie niepotrzebne;
import warnings
warnings.filterwarnings("ignore")
#czytamy wszystkie dane w bazie danych i tworzymy bazowy dataframe df
engine = sqlalchemy.create_engine('sqlite:///clinical_res.db')
#connection = sqlite3.connect("testdb.db")
#sql_query = "select * from entries"
df = pd.read_sql_table('entries', engine)
print(f"Got dataframe with {len(df)} entries")
pd.read_sql(
        "SELECT * FROM entries", engine)
        #zamieniamy dane string na numeryczne
df.loc[df["start_month"] == "January", "start_month"] = 1
df.loc[df["start_month"] == "February", "start_month"] = 2
df.loc[df["start_month"] == "March", "start_month"] = 3
df.loc[df["start_month"] == "April", "start_month"] = 4
df.loc[df["start_month"] == "May", "start_month"] = 5
df.loc[df["start_month"] == "June", "start_month"] = 6
df.loc[df["start_month"] == "July", "start_month"] = 7
df.loc[df["start_month"] == "August", "start_month"] = 8
df.loc[df["start_month"] == "September", "start_month"] = 9
df.loc[df["start_month"] == "October", "start_month"] = 10
df.loc[df["start_month"] == "November", "start_month"] = 11
df.loc[df["start_month"] == "December", "start_month"] = 12
df['start_month'] = df['start_month'].astype(int)
df['end_month'] = df['end_month'].astype(int)
#budujemy dataframe z samymi rekordami lat rozpoczęcia projektów
data = pd.DataFrame(df, columns=['start_year'])
#odrzucamy lata przed 2020
reference_period = df.loc[(df['start_year'] > 1999)]
plot = reference_period.groupby(reference_period['start_year']).count()
#sprawdzamy jak teraz wygląda dataframe
reference_period.describe()
#nie interesuje nas kategoria Early Phase 1 w kolumnie phase - dlatego łączymy ją z Phase 1
df['phase'] = df['phase'].apply(lambda x: x.replace("Early Phase 1", "Phase 1"))
#wyrysowujemy wykres projektów w roku
ax = sns.barplot(plot, y=plot['id'], x=plot.index, palette="dark:salmon_r")
ax.set(xlabel='rok', ylabel='liczba rozpoczętych projektów', title="Liczba projektów badawczych wykonanych w okresie 2001-2023\ndane zebrane w lutym 2023r.")
ax.xaxis.set_major_locator(ticker.MultipleLocator(1))
ax.figure.set_size_inches(12, 6)
plt.tight_layout()
# w tym momencie warto przypomnieć, że analizujemy tylko projekty ze statusem "completed"
#liczymy medianę, q1 i q3 dla długości projektów wg. faz po roku 2000, ogólnie
temp_df = df.groupby('phase')

median_temp_series = temp_df.quantile(0.5)['months_spent']
P25_temp_series = temp_df.quantile(0.25)['months_spent']
P75_temp_series = temp_df.quantile(0.75)['months_spent']

#łączymy serie w jeden dataframe
plot_df = pd.concat([P25_temp_series, median_temp_series, P75_temp_series], keys=['pierwszy kwartyl', 'mediana', 'trzeci kwartyl'], axis=1)

plot_df.plot(kind="bar", title="Czas trwania fazy badawczej w miesiącach, okres 2000-2023", xlabel = "Faza", ylabel="Czas trwania w miesiącach", figsize=(10, 8), colormap='cividis')
<AxesSubplot: title={'center': 'Czas trwania fazy badawczej w miesiącach, okres 2000-2023'}, xlabel='Faza', ylabel='Czas trwania w miesiącach'>
#powyżej widać, że czas trwania projektów jest nierównomierny. Faza pierwsza trwa najkrócej. Projekty, które łączą fazę 1 z 2 są najdłuższe, Prawidłowość zachodzi we wszystkich kwartylach (tj. zarówno dla najkrótszych, jak i najdłuższych projektów w danej fazie)
#rozdzielamy dane na przed pandemią i pandemię;
#zakładamy, że okres pandemii zazczął się w grudniu 2019 roku

#ponownie łączymy kategorie (pracujemy teraz na innym dataframe niż poprzednio)
reference_period['phase'] = reference_period['phase'].apply(lambda x: x.replace("Early Phase 1", "Phase 1"))
pre_covid_df = reference_period[(reference_period['start_year']<2019) | ((reference_period['start_year']==2019) & (reference_period['start_month']<12))]
covid_df = reference_period[(reference_period['start_year']>2019) | ((reference_period['start_year']==2019) & (reference_period['start_month']>11))]
pre_covid_df #dataframe z danymi sprzed pandemii - sprawdzamy czy filtr działa dobrze
#wygląda na to, że wszystko jest ok
covid_df_group_by = covid_df.groupby('phase')
median_covid_series = covid_df_group_by.quantile(0.5)['months_spent']
P25_covid_series = covid_df_group_by.quantile(0.25)['months_spent']
P75_covid_series = covid_df_group_by.quantile(0.75)['months_spent']

pre_covid_df_group_by = pre_covid_df.groupby('phase')
median_pre_covid_series = pre_covid_df_group_by.quantile(0.5)['months_spent']
P25_pre_covid_series = pre_covid_df_group_by.quantile(0.25)['months_spent']
P75_pre_covid_series = pre_covid_df_group_by.quantile(0.75)['months_spent']
#generujemy wykres dla badań wykonywanych w trakcie pandemii
plot_df = pd.concat([P25_pre_covid_series,median_pre_covid_series,P75_pre_covid_series, P25_covid_series,  median_covid_series,  P75_covid_series ], keys=[ 'Przed COVID: pierwszy kwartyl', 'Przed COVID: mediana', 'Przed COVID: trzeci kwartyl', 'COVID: pierwszy kwartyl','COVID: mediana', 'COVID: trzeci kwartyl' ], axis=1)
plot_df.plot(kind="bar", title="Czas trwania fazy badawczej w miesiącach w trakcie pandemii COVID-19", xlabel = "Faza", ylabel="Czas trwania w miesiącach", figsize=(10, 8), colormap='Accent')
<AxesSubplot: title={'center': 'Czas trwania fazy badawczej w miesiącach w trakcie pandemii COVID-19'}, xlabel='Faza', ylabel='Czas trwania w miesiącach'>
#powyższy wykres rozróżnia czas trwania projektów badawczych w podziale na fazy kwartyle w okresie przed pandemią i po ogłoszeniu pandemii COVID-19. Okazuje się, że że czas trwania faz uległ drastycznemu skróceniu w każdej fazie. Przykładowo, mediana trwania projektów fazy 1 spadła z ok. 13 miesięcy do ok. 6;
reference_period['covid_condition'] = reference_period['condition'].str.contains(r'COVID')
covid_research_df = reference_period[reference_period['covid_condition']==True]
print("W bazie mamy: " + str(len(covid_research_df)) + " projektów, które dotczą zakresem COVID")

groupby_reference = covid_research_df.groupby('start_year')

temp = groupby_reference.count()['id']
print(temp)
fig = plt.figure()
ax = fig.add_axes([0,0,1,1])
ax.xaxis.set_major_locator(ticker.MultipleLocator(1))
ax.grid(False)
plt.ylim(bottom = 1, top=360 )
ax.set_title('Projekty, które obejmowały zakresem COVID')

ax.bar(temp.index, temp+5, color='green',lw=2)
plt.tight_layout()
#uwaga, podnosimy o 5 wartość wyrysowywaną na wykresie żeby jeden projekt z 2018r był widoczny bez podnoszenia dpi
W bazie mamy: 489 projektów, które dotczą zakresem COVID
start_year
2018      1
2020    353
2021    110
2022     25
Name: id, dtype: int64

# co ciekawe, jeden projekt z roku 2018 ma w zakresie COVID-19, co może sugerować, że zakres projektu zmieniał się wraz z upływem czasu
# badania inne niż na COVID 19 
reference_period['covid_condition'] = reference_period['condition'].str.contains(r'COVID')
covid_research_df = reference_period[reference_period['covid_condition']==False]
print("W bazie mamy: " + str(len(covid_research_df)) + " projektów, które dotczą zakresem COVID")

groupby_reference = covid_research_df.groupby('start_year')

temp = groupby_reference.count()['id']
print(temp)
fig = plt.figure()
ax = fig.add_axes([0,0,1,1])
ax.xaxis.set_major_locator(ticker.MultipleLocator(1))
ax.grid(False)
plt.ylim(bottom = 1, top=7000 )
ax.set_title('Projekty, które obejmowały zakresem COVID')

ax.bar(temp.index, temp+0, color='green',lw=2)
plt.tight_layout()
W bazie mamy: 96540 projektów, które dotczą zakresem COVID
start_year
2000    1096
2001    1464
2002    2205
2003    2958
2004    3837
2005    4535
2006    5129
2007    5498
2008    5994
2009    6262
2010    6226
2011    6238
2012    5990
2013    5799
2014    5896
2015    5629
2016    5122
2017    4648
2018    4040
2019    3280
2020    2431
2021    1774
2022     488
2023       1
Name: id, dtype: int64

reference_period['behavioral_intervention'] = reference_period['intervention_type'].str.contains(r'Behavioral')
reference_period['drug_intervention'] = reference_period['intervention_type'].str.contains(r'Drug')
reference_period['procedure_intervention'] = reference_period['intervention_type'].str.contains(r'Procedure')
reference_period['biological_intervention'] = reference_period['intervention_type'].str.contains(r'Biological')
reference_period['device_intervention'] = reference_period['intervention_type'].str.contains(r'Device')
reference_period['dietary_intervention'] = reference_period['intervention_type'].str.contains(r'Dietary')
reference_period['genetic_intervention'] = reference_period['intervention_type'].str.contains(r'Genetic')
reference_period['radiation_intervention'] = reference_period['intervention_type'].str.contains(r'Radiation')
reference_period['combination_product_intervention'] = reference_period['intervention_type'].str.contains(r'Combination')
reference_period['diagnostic_intervention'] = reference_period['intervention_type'].str.contains(r'Diagnostic')
reference_period['other_intervention'] = reference_period['intervention_type'].str.contains(r'Other')
temp_df = reference_period[['drug_intervention', 'other_intervention', 'procedure_intervention', 'behavioral_intervention', 'radiation_intervention', 'biological_intervention', 'dietary_intervention', 'genetic_intervention', 'device_intervention', 'combination_product_intervention', 'diagnostic_intervention', 'intervention_type', 'id']]
temp_df.replace({False: 0, True: 1}, inplace=True)
temp_df['sum_row'] =  temp_df.sum(axis=1)
temp_df
drug_intervention	other_intervention	procedure_intervention	behavioral_intervention	radiation_intervention	biological_intervention	dietary_intervention	genetic_intervention	device_intervention	combination_product_intervention	diagnostic_intervention	intervention_type	id	sum_row
81	1	0	0	0	0	0	0	0	0	0	0	Drug: Buprenorphine;	NCT00000299	1
84	1	0	0	0	0	0	0	0	0	0	0	Drug: Naltrexone;	NCT00000307	1
98	1	0	0	0	0	0	0	0	0	0	0	Drug: Benztropine;	NCT00000333	1
99	1	0	0	0	0	0	0	0	0	0	0	Drug: Buprenorphine;	NCT00000334	1
140	0	1	1	0	0	0	0	0	0	0	0	Procedure: Decompressive laminectomy; Other: N...	NCT00000409	2
...	...	...	...	...	...	...	...	...	...	...	...	...	...	...
99986	1	0	0	0	0	0	0	0	0	0	0	Drug: Glargine and regular insulin versus NPH ...	NCT05709938	1
99987	1	0	0	0	0	0	0	0	1	0	0	Drug: SoftOx Biofilm Eradicator (Groups 1 to 7...	NCT05710094	2
99988	1	0	0	1	0	0	1	0	0	0	1	Drug: Nifedipine 30 MG; Diagnostic Test: Kangz...	NCT05711810	4
99989	0	0	0	0	0	1	0	0	0	0	0	Biological: Stem cell implantation;	NCT05712148	1
99990	1	0	0	0	0	0	0	0	0	0	0	Drug: Intravitreal Bevacizumab;	NCT05712642	1
97029 rows × 14 columns

#sprawdzamy czy istnieją jakieś wiersze, które nie mają typu interwencji
temp_df.loc[temp_df['sum_row'] ==  0]
drug_intervention	other_intervention	procedure_intervention	behavioral_intervention	radiation_intervention	biological_intervention	dietary_intervention	genetic_intervention	device_intervention	combination_product_intervention	diagnostic_intervention	intervention_type	id	sum_row
2316	0	0	0	0	0	0	0	0	0	0	0		NCT00018629	0
#okazuje się, że istnieje jeden taki wiersz, odrzucamy go przez filtrowanie po id

temp_df = temp_df[temp_df.sum_row>0] #odrzucamy też z temp_df
### tutaj będziemy modyfikowac sobie lata na których pracujemy > 2019 lub <2019 
reference_period = df.loc[(df['start_year'] > 1999)]
behavioral_gb = reference_period.groupby('behavioral_intervention')
behavioral_df = behavioral_gb.count()['id']
behavioural_share = (behavioral_df / behavioral_df.sum()) * 100
#res_df = res_df.apply(lambda x: x / max)

print(behavioural_share)
behavioural_share.plot(kind='pie', labels=['nie zastosowano leczenia\n behawioralnego', 'zastosowano leczenie\nbehawioralne'], autopct='%.0f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów, które stosowały leczenie behawioralne", ylabel="")
behavioral_intervention
False    96.665945
True      3.334055
Name: id, dtype: float64
<AxesSubplot: title={'center': 'Procent projektów, które stosowały leczenie behawioralne'}>

 
res_df_gb = reference_period.groupby('drug_intervention')
drug_intervention = res_df_gb.count()['id']
drug_share = (drug_intervention / drug_intervention.sum()) * 100
drug_share.plot(kind='pie', labels=['nie zastosowano leczenia\npreparatem chemicznym', 'zastosowano leczenie\npreparatem chemicznym'], autopct='%.0f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów, które stosowały leczenie preparatem chemicznym", ylabel="")
<AxesSubplot: title={'center': 'Procent projektów, które stosowały leczenie preparatem chemicznym'}>

res_df_gb = reference_period.groupby('procedure_intervention')
procedure_intervention = res_df_gb.count()['id']
procedure_intervention

procedure_share = (procedure_intervention / procedure_intervention.sum()) * 100
#res_df = res_df.apply(lambda x: x / max)

print(procedure_intervention)
procedure_intervention.plot(kind='pie', labels=['nie zastosowano leczenia\nzabiegiem', 'zastosowano leczenie\nzabiegiem'], autopct='%.0f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów, które stosowały leczenie zabiegiem", ylabel="")
procedure_intervention
False    92226
True      4803
Name: id, dtype: int64
<AxesSubplot: title={'center': 'Procent projektów, które stosowały leczenie zabiegiem'}>

res_df_gb = reference_period.groupby('biological_intervention')
biological_intervention = res_df_gb.count()['id']
biological_share = (biological_intervention / biological_intervention.sum()) * 100
biological_share.plot(kind='pie', labels=['nie zastosowano leczenia\n biologicznego', 'zastosowano leczenie\nbiologiczne'], autopct='%.0f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów, które stosowały leczenie preparatem biologicznym", ylabel="")
<AxesSubplot: title={'center': 'Procent projektów, które stosowały leczenie preparatem biologicznym'}>

res_df_gb = reference_period.groupby('device_intervention')
device_intervention = res_df_gb.count()['id']
print(device_intervention)
device_share = (biological_intervention / biological_intervention.sum()) * 100
device_share.plot(kind='pie', labels=['nie zastosowano leczenia\nurządzeniem', 'zastosowano leczenie\nurządzeniem'], autopct='%.0f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów, które stosowały leczenie urządzeniem", ylabel="")
device_intervention
False    92485
True      4544
Name: id, dtype: int64
<AxesSubplot: title={'center': 'Procent projektów, które stosowały leczenie urządzeniem'}>

res_df_gb = reference_period.groupby('dietary_intervention')
dietary_intervention = res_df_gb.count()['id']
dietary_share = (dietary_intervention / dietary_intervention.sum()) * 100
dietary_share.plot(kind='pie', labels=['nie zastosowano leczenia\n dietą', 'zastosowano leczenie\ndietą'], autopct='%.0f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów, które stosowały leczenie dietą", ylabel="")
<AxesSubplot: title={'center': 'Procent projektów, które stosowały leczenie dietą'}>

res_df_gb = reference_period.groupby('genetic_intervention')
genetic_intervention = res_df_gb.count()['id']
print(genetic_intervention)
genetic_share = (genetic_intervention / genetic_intervention.sum()) * 100
genetic_share.plot(kind='pie', labels=['Nie zastosowano metod genetycznych', 'Zastosowano \nmetody genetyczne'], autopct='%.00001f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów, które stosowały metody genetyczne", ylabel="")
genetic_intervention
False    96733
True       296
Name: id, dtype: int64
<AxesSubplot: title={'center': 'Procent projektów, które stosowały metody genetyczne'}>

res_df_gb = reference_period.groupby('combination_product_intervention')
combination_intervention = res_df_gb.count()['id']
print(combination_intervention)
combination_share = (combination_intervention / combination_intervention.sum()) * 100
combination_share.plot(kind='pie', labels=['Nie zastosowano metod kombinacyjnych', 'Zastosowano \nmetody kombinacyjne'], autopct='%.00001f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów, które stosowały metody kombinacyjne", ylabel="")
combination_product_intervention
False    96491
True       538
Name: id, dtype: int64
<AxesSubplot: title={'center': 'Procent projektów, które stosowały metody kombinacyjne'}>

res_df_gb = reference_period.groupby('diagnostic_intervention')
diagnostic_intervention = res_df_gb.count()['id']
print(diagnostic_intervention)
diagnostic_share = (combination_intervention / combination_intervention.sum()) * 100
diagnostic_share.plot(kind='pie', labels=['Projekty nie-diagnostyczne', 'Projekty diagnostyczne'], autopct='%.00001f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów diagnostycznych", ylabel="")
diagnostic_intervention
False    96882
True       147
Name: id, dtype: int64
<AxesSubplot: title={'center': 'Procent projektów diagnostycznych'}>

res_df_gb = reference_period.groupby('other_intervention')
other_intervention = res_df_gb.count()['id']
other_share = (other_intervention / other_intervention.sum()) * 100
other_share.plot(kind='pie', labels=['Zastosowano standardowe\nmetody leczenia', 'zastosowano inny\nsposób leczenia'], autopct='%.0f%%', shadow=True, radius=0.5, figsize=(10,10), title="Procent projektów, które stosowały inne niż najpopularniejsze metody (behawioralne, medykamentem, zabiegowe, dietetyczne, biologiczne, genetyczne, radiacyjne, kombinowane, diagnostyczne)", ylabel="")
<AxesSubplot: title={'center': 'Procent projektów, które stosowały inne niż najpopularniejsze metody (behawioralne, medykamentem, zabiegowe, dietetyczne, biologiczne, genetyczne, radiacyjne, kombinowane, diagnostyczne)'}>

# znów modyfikujemy lata przy pomocy reference_period = df.loc[(df['start_year'] > .....)]

lead_sp_gr = reference_period.groupby('lead_sponsor')
lead_sp = lead_sp_gr.count()
lead_sp = lead_sp.sort_values(['id'], ascending=False)
top10_lead = lead_sp.head(10)['id']
top10_lead.plot(kind='bar', xlabel= "nazwa firmy", ylabel="liczba prowadzonych projektów", title="Leaderzy projektów w latach 2000-2023")
<AxesSubplot: title={'center': 'Leaderzy projektów w latach 2000-2023'}, xlabel='nazwa firmy', ylabel='liczba prowadzonych projektów'>

df2 = pd.read_sql(
        """Select lead_sponsor, phase, count(phase) from entries where lead_sponsor in ('GlaxoSmithKline',
        'Pfizer',
        'AstraZeneca',
        'Eli Lilly and Company',
        'Novartis Pharmaceuticals',
        'Merck Sharp & Dohme LLC',
        'Hoffmann-La Roche',
        'National Cancer Institute (NCI)',
        'Sanofi',
        'Bayer')
group by phase, lead_sponsor ORDER BY lead_sponsor ASC""", engine)
#tym razem pozostawiamy early phase1
company_list = ['GlaxoSmithKline',
                'Pfizer',
                'AstraZeneca',
                'Eli Lilly and Company',
                'Novartis Pharmaceuticals',
                'Merck Sharp & Dohme LLC',
                'Hoffmann-La Roche',
                'National Cancer Institute (NCI)',
                'Sanofi',
                'Bayer']

for company in company_list :
        temp_plot_df = df2[(df2['lead_sponsor'] == company)]
        temp_plot_df.drop(['lead_sponsor'], axis=1, inplace=True)
        temp_plot_df.set_index(['phase'])
        title_string = "Rozkład projektów według fazy dla firmy: " + company
        temp_plot_df.plot(x = 'phase',y='count(phase)', kind='bar', title = title_string, legend=None, ylabel='liczba wykonanych projektów', xlabel='faza')
        #temp_plot_df.plot(df2['phase'], df2['count(phase)'], title = company, kind = 'bar')










 
