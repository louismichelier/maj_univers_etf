But du nouveau code : 

Procédure de mise à jour :
Prendre les ISIN MEMBer du Stoxx 600 (SSXP Index ou .SXXP)
Prendre les ISIN des Sous-Jacent Convertibles
Prendre les RIC des sous-jacents de l'indice via data Refinitiv
Les remplir dans l'onglet Data (formule)
Ensuite, faire un copier coller en valeur sur l'onglet Data
Corriger quelques données manuelle si besoin

Ensuite mettre à jour les fichier FDS avec code ISIN


# maj_univers_etf


import pandas as pd
from datetime import datetime
from xbbg import blp

# ========= PARAMETRES =========
parquet_path = "prices_long_px_last.parquet"
output_parquet = "prices_long_px_last.parquet"
output_excel = "prices_long_px_last.xlsx"

# ========= MAPPING COLONNES FICHIER -> FLDS BLOOMBERG =========
col_to_bbg = {
    "PX_LAST": "PX_LAST",
    "OPEN": "OPEN",
    "HIGH": "HIGH",
    "LOW": "LOW",
    "VOLUME": "VOLUME",
}

# ========= LECTURE =========
df = pd.read_parquet(parquet_path)
df.columns = [str(c).strip().upper() for c in df.columns]

if "DATE" not in df.columns or "TICKER" not in df.columns:
    raise ValueError("Le parquet doit contenir au minimum DATE et TICKER")

df["DATE"] = pd.to_datetime(df["DATE"], errors="coerce")
df = df.dropna(subset=["DATE", "TICKER"]).copy()

# ========= CONTROLES =========
last_date = df["DATE"].max()
tickers = sorted(df["TICKER"].astype(str).str.strip().dropna().unique().tolist())

data_cols = [c for c in df.columns if c not in ["DATE", "TICKER"]]
requested_cols = [c for c in data_cols if c in col_to_bbg]
flds_bloom = [col_to_bbg[c] for c in requested_cols]

print("Dernière date du fichier :", last_date.date())
print("Nombre de tickers :", len(tickers))
print("Liste des tickers :")
print(tickers)

print("\nColonnes présentes dans le fichier :")
print(df.columns.tolist())

print("\nColonnes de données appelées dans Bloomberg :")
print(requested_cols)

print("\nFLDS Bloomberg envoyés à blp.bdh() :")
print(flds_bloom)

print("\nDate de fin de mise à jour :")
print(datetime.today().strftime("%Y-%m-%d"))


#2e cellule

from datetime import timedelta

# ========= PLAGE DE MISE A JOUR =========
start_date = last_date.strftime("%Y-%m-%d")
end_date = datetime.today().strftime("%Y-%m-%d")

# ========= HISTORIQUE SANS LE DERNIER JOUR =========
df_hist = df[df["DATE"] < last_date].copy()

# ========= APPEL BLOOMBERG =========
raw = blp.bdh(
    tickers=tickers,
    flds=flds_bloom,
    start_date=start_date,
    end_date=end_date,
    adjust="all"   # retire cette ligne si tu veux du non-ajusté
)

#convert to df
#comparer les versions de blp api avec LB ==> download la mm que LB (appel avec ZORGUI Jérémy)

if raw.empty:
    print("Aucune donnée Bloomberg récupérée.")
    df_final = (
        df.sort_values(["DATE", "TICKER"], ascending=[True, True])
          .reset_index(drop=True)
    )
else:
    raw.columns.names = ["TICKER", "FIELD"]

    df_new = (
        raw
        .stack("TICKER")
        .reset_index()
    )

    # Renommer la première colonne en DATE si nécessaire
    first_col = df_new.columns[0]
    if first_col != "DATE":
        df_new = df_new.rename(columns={first_col: "DATE"})

    # ========= CLÉ DU PROBLEME : normaliser en MAJUSCULES =========
    # Bloomberg renvoie 'px_last', 'date', 'ticker' en minuscules
    df_new.columns = [str(c).strip().upper() for c in df_new.columns]
    # ==============================================================

    print("\n===== COLONNES APRES NORMALISATION =====")
    print(df_new.columns.tolist())
    print(df_new.head())

    # Garder les colonnes attendues
    final_cols = ["DATE", "TICKER"] + requested_cols
    existing_cols = [c for c in final_cols if c in df_new.columns]
    df_new = df_new[existing_cols].copy()

    df_new["DATE"] = pd.to_datetime(df_new["DATE"], errors="coerce")
    df_new = df_new.dropna(subset=["DATE", "TICKER"]).copy()

    print("\n===== DF_NEW APRES FILTRE =====")
    print(df_new.tail(10))
    print(df_new.isna().sum())

    df_final = pd.concat([df_hist, df_new], ignore_index=True, sort=False)

    df_final = (
        df_final
        .sort_values(["DATE", "TICKER"], ascending=[True, True])
        .drop_duplicates(subset=["DATE", "TICKER"], keep="last")
        .reset_index(drop=True)
    )


# ========= EXPORT =========
df_final.to_parquet(output_parquet, index=False)
df_final.to_excel(output_excel, index=False)

print("Mise à jour terminée.")
print(df_final.tail())

-
voiici les fichiers avec lesquels je travaille 
Fichier alloc univers : 
3 onglets : 
1) Description : Colonne	Type	Description									
ISIN	string										
TICKER_BBG	string										Fichiers qui pointe sur cette liste
RIC	string										Publications trimestrielles EU.BDP.xlsm
NAME	string										Univers EU Facset
WEIGHT STOXX 600	float										
POIDS SX5E	float										
SHARES	float										
PRICE	float										Procédure de mise à jour :
											Prendre les ISIN MEMBer du Stoxx 600 (SSXP Index ou .SXXP)
											Prendre les ISIN des Sous-Jacent Convertibles
											Prendre les RIC des sous-jacents de l'indice via data Refinitiv
											Les remplir dans l'onglet Data (formule)
											Ensuite, faire un copier coller en valeur sur l'onglet Data
Ajout le 	01/06/2026	Daté émission et maturité du titre									Corriger quelques données manuelle si besoin
	11/06/2026	Ticker BBG pour CDS 5y									
											Ensuite mettre à jour les fichier FDS avec code ISIN

2)   Data : ID_ISIN	EQY_FUND_TICKER	1	P_SYMBOL	FSYM_ENTITY_ID		SECURITY_NAME	98,057502						TR.ICBIndustry	TR.ICBSuperSector	TR.CorCompanyCountryCode	TR.CorCompanyCountry		Last MàJ	22/05/2026		TR.FirstTradeDate	HSTCLSDATE		TR.CDSPrimaryCDSRic	CDS_SPREAD_TICKER_5Y	ID_BB_COMPANY		30/06/2026
ISIN	TICKER_BBG	RIC	FACTSET_ID	FACTSET_ENTITY_ID	INVESTISSABLE	NAME	WEIGHT_STOXX_600	TICKER_BBG_INDEX	RIC_INDEX	HYPER_SECTOR	TICKER_BBG_SECTOR_L2	RIC_SECTOR_L2	SECTOR_L1_NAME	SECTOR_L2_NAME	PAYS_ISO	PAYS_NAME	POIDS_SX5E	SHARES	PRICE	DATE_INSERTION	DATE_EMISSION	DATE_ECHEANCE	TR.F.AvgNumShrOutstIssue(Period=FY0)	CDS_RIC	CDS_SPREAD_5Y_TICKER_BBG	ID_BB_COMPANY	TP_N	TP_N+1

3) Data Formula : ID_ISIN	EQY_FUND_TICKER	0	P_SYMBOL	FSYM_ENTITY_ID		SECURITY_NAME										TR.ICBIndustry	TR.ICBSuperSector	TR.CorCompanyCountryCode	TR.CorCompanyCountry		Last MàJ	22/05/2026		TR.FirstTradeDate	HSTCLSDATE		TR.CDSPrimaryCDSRic	CDS_SPREAD_TICKER_5Y	ID_BB_COMPANY		30/06/2026	
ISIN	TICKER_BBG	RIC	FACTSET_ID	FACTSET_ENTITY_ID	INVESTISSABLE	NAME	ACTIF_1	ACTIF_2	GEOGRAPHIE_1	GEOGRAPHIE_2	TICKER_BBG_INDEX	RIC_INDEX	HYPER_SECTOR	TICKER_BBG_SECTOR_L2	RIC_SECTOR_L2	SECTOR_L1_NAME	SECTOR_L2_NAME	PAYS_ISO	PAYS_NAME	POIDS_SX5E	SHARES	PRICE	DATE_INSERTION	DATE_EMISSION	DATE_ECHEANCE	TR.F.AvgNumShrOutstIssue(Period=FY0)	CDS_RIC	CDS_SPREAD_5Y_TICKER_BBG	ID_BB_COMPANY			
NL0010273215	ASML NA Equity	#NOM?	#NOM?	#NOM?	0	ASML Holding NV					SXXP Index	.SXXP	#REF!	#REF!	#REF!	#NOM?	Technology	NL	Netherlands		388,147674	1151,8	02/04/2026	#NOM?	04/09/2026		0	CT426050	163205			
ID_ISIN	EQY_FUND_TICKER	0	P_SYMBOL	FSYM_ENTITY_ID		SECURITY_NAME										TR.ICBIndustry	TR.ICBSuperSector	TR.CorCompanyCountryCode	TR.CorCompanyCountry		Last MàJ	22/05/2026		TR.FirstTradeDate	HSTCLSDATE		TR.CDSPrimaryCDSRic	CDS_SPREAD_TICKER_5Y	ID_BB_COMPANY		30/06/2026	
ISIN	TICKER_BBG	RIC	FACTSET_ID	FACTSET_ENTITY_ID	INVESTISSABLE	NAME	ACTIF_1	ACTIF_2	GEOGRAPHIE_1	GEOGRAPHIE_2	TICKER_BBG_INDEX	RIC_INDEX	HYPER_SECTOR	TICKER_BBG_SECTOR_L2	RIC_SECTOR_L2	SECTOR_L1_NAME	SECTOR_L2_NAME	PAYS_ISO	PAYS_NAME	POIDS_SX5E	SHARES	PRICE	DATE_INSERTION	DATE_EMISSION	DATE_ECHEANCE	TR.F.AvgNumShrOutstIssue(Period=FY0)	CDS_RIC	CDS_SPREAD_5Y_TICKER_BBG	ID_BB_COMPANY			
NL0010273215	ASML NA Equity	#NOM?	#NOM?	#NOM?	0	ASML Holding NV					SXXP Index	.SXXP	#REF!	#REF!	#REF!	#NOM?	Technology	NL	Netherlands		388,147674	1151,8	02/04/2026	#NOM?	04/09/2026		0	CT426050	163205			

