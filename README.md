# E-Commerce SQL Analytics — Olist Dataset

## Overview
End-to-end SQL analytics project on the Brazilian E-Commerce (Olist) dataset
from Kaggle — 96,477 real orders across a 9-table relational schema.

## Tech Stack
MySQL 8.0 | MySQL Workbench

## Queries Covered
| # | Query | Concepts Used |
|---|-------|---------------|
| Q1 | Monthly Revenue Trend | DATE_FORMAT, SUM, GROUP BY |
| Q2 | Order Count by Status | CASE WHEN, window % |
| Q3 | Top 10 Categories by Revenue | 3-table JOIN |
| Q4 | Customers by State | COUNT DISTINCT, HAVING |
| Q5 | Avg Delivery Time by State | TIMESTAMPDIFF, AVG |
| Q6 | Payment Method Breakdown | GROUP BY, AVG |
| Q7 | Seller Performance Ranking | RANK() OVER() |
| Q8 | Review Score vs Delivery Delay | CASE WHEN buckets |
| Q9 | Seller Location vs Delivery Speed | 4-table JOIN |
| Q10 | Cumulative Revenue + MoM Growth | LAG(), SUM() OVER() |
| Q13 | RFM Customer Segmentation | NTILE(5), 3-CTE chain |

## Key Findings
- Peak month: Nov 2017 — 7,289 orders, R$1.15M revenue
- Delivery gap: SP avg 8.7 days vs RR avg 29.6 days across 27 states
- Review score drops from 4.31 stars (on-time) to 2.66 stars (7+ days late)
- RFM segmentation: 93,357 customers across 7 segments —
  Loyal Customers drove highest revenue at R$3.38M

## Dataset
https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce
