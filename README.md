# algo-mini-project
réaliser une application concrète : une Facture Unique de services (eau, gaz, électricité, téléphone et autres),

/*
 * ================================================================
 *                      FACTURE UNIQUE
 *             Multi-Utility Billing System
 * ================================================================
 *
 * File        : bill.h
 * Project     : ASDS Mini Project - ESTIN
 * Description : Header file for the billing system.
 * Contains    : Constants, data structures and function prototypes.
 *
 * ================================================================
 */

#ifndef BILL_H
#define BILL_H

#include <stdio.h>

/* ================================================================
 * 01. TARIFF CONFIGURATION
 * ================================================================
 * All prices and tax rates are defined according to Annex A.
 */

/* ---------- ELECTRICITY ---------- */
#define ELEC_LIMIT1       100.0
#define ELEC_LIMIT2       300.0
#define ELEC_PRICE1         2.0
#define ELEC_PRICE2         4.0
#define ELEC_PRICE3         6.0
#define ELEC_FIXED_FEE    100.0
#define ELEC_VAT            0.09

/* ---------- NATURAL GAS ---------- */
#define GAS_LIMIT1        100.0
#define GAS_LIMIT2        300.0
#define GAS_PRICE1          1.5
#define GAS_PRICE2          3.0
#define GAS_PRICE3          4.5
#define GAS_FIXED_FEE     50.0
#define GAS_VAT             0.09

/* ---------- WATER ---------- */
#define WATER_LIMIT1       20.0
#define WATER_LIMIT2       40.0
#define WATER_PRICE1        6.0
#define WATER_PRICE2       20.0
#define WATER_PRICE3       40.0
#define WATER_SANITATION    0.10
#define WATER_FIXED_FEE   100.0
#define WATER_VAT           0.09

/* ---------- TELEPHONE & INTERNET ---------- */
#define PHONE_SUBSCRIPTION 600.0
#define PHONE_INCLUDED_MIN 100.0
#define PHONE_EXTRA_PRICE    3.0

#define NET_PRICE_4MB     1000.0
#define NET_PRICE_8MB     1500.0
#define NET_PRICE_20MB    2500.0

#define PHONE_VAT            0.19

/* ---------- ADDITIONAL SERVICE: WASTE COLLECTION ---------- */
#define WASTE_FIXED_FEE   150.0
#define WASTE_VAT           0.09


/* ================================================================
 * 02. GENERAL CONFIGURATION
 * ================================================================
 */

#define MAX_LINES  5
#define STR_LEN   50


/* ================================================================
 * 03. DATA STRUCTURES
 * ================================================================
 */

/* Information about one billed service */
typedef struct {
    char   name[STR_LEN];

    int    has_index;
    double previous;
    double current;
    double consumption;

    double before_tax;
    double vat;
    double total;

} BillLine;


/* Complete customer invoice */
typedef struct {
    char     customer[STR_LEN];
    char     account[STR_LEN];
    char     period[STR_LEN];

    BillLine lines[MAX_LINES];
    int      count;

    double   total_before_tax;
    double   total_vat;
    double   total_to_pay;

} Bill;


/* ================================================================
 * 04. BILLING & CALCULATION FUNCTIONS
 * ================================================================
 */

/* Progressive three-tier tariff calculation */
double progressive_amount(double cons, double lim1, double lim2,
                          double p1, double p2, double p3);

/* Utility cost calculations */
double electricity_amount(double kwh);
double gas_amount(double m3);
double water_amount(double m3);

/* Communication service calculations */
double phone_amount(double minutes);
double internet_amount(int mbps);

/* Tax calculation */
double compute_vat(double amount, double rate);


/* ================================================================
 * 05. INVOICE MANAGEMENT
 * ================================================================
 */

/* Initialize a new invoice */
void bill_init(Bill *b);

/* Add a service to the invoice */
void bill_add_line(Bill *b, const char *name, int has_index,
                   double prev, double cur,
                   double before_tax, double vat_rate);

/* Calculate invoice totals */
void bill_compute_totals(Bill *b);

/* Display or export the invoice */
void display_bill(const Bill *b, FILE *out);


/* ================================================================
 * 06. INPUT & VALIDATION
 * ================================================================
 */

/* Read and validate user input */
double read_double(const char *prompt, double min);
int    read_int(const char *prompt, int min, int max);
void   read_string(const char *prompt, char *buf, int size);
int    read_yes_no(const char *prompt);


/* ================================================================
 *                    END OF BILL.H
 * ================================================================
 */

#endif /* BILL_H */
