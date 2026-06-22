# sales-analysis
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.ticker as mtick
import seaborn as sns
import warnings

warnings.filterwarnings('ignore')

# ── Styling ──────────────────────────────────────────────────
sns.set_theme(style='whitegrid', palette='muted')
plt.rcParams['figure.dpi'] = 130

PALETTE = {
    'Technology':       '#378ADD',
    'Furniture':        '#EF9F27',
    'Office Supplies':  '#1D9E75'
}

# ── 1. LOAD DATA ─────────────────────────────────────────────
print("Loading data...")
df = pd.read_csv('superstore.csv', encoding='latin-1')
print(f"  Rows: {len(df)}  |  Columns: {len(df.columns)}")

# ── 2. CLEAN DATA ────────────────────────────────────────────
print("Cleaning data...")

# Parse dates if columns exist
if 'Order Date' in df.columns:
    df['Order Date'] = pd.to_datetime(df['Order Date'])
    df['Month']      = df['Order Date'].dt.month
    df['Month Name'] = df['Order Date'].dt.strftime('%b')
    df['Year']       = df['Order Date'].dt.year

# Remove duplicates
before = len(df)
df.drop_duplicates(inplace=True)
print(f"  Removed {before - len(df)} duplicates. Final rows: {len(df)}")

# Check nulls
nulls = df.isnull().sum().sum()
print(f"  Null values: {nulls}")

# ── 3. SUMMARY STATS ─────────────────────────────────────────
print("\n=== Summary Statistics ===")
print(df[['Sales', 'Profit', 'Quantity', 'Discount']].describe().round(2))

print("\n=== Sales by Region ===")
print(df.groupby('Region')['Sales'].sum().sort_values(ascending=False).apply(lambda x: f"${x:,.0f}"))

print("\n=== Sales by Category ===")
print(df.groupby('Category')['Sales'].sum().sort_values(ascending=False).apply(lambda x: f"${x:,.0f}"))

# ── 4. BUILD DASHBOARD ───────────────────────────────────────
print("\nGenerating charts...")

MONTH_ORDER = ['Jan','Feb','Mar','Apr','May','Jun',
               'Jul','Aug','Sep','Oct','Nov','Dec']

fig = plt.figure(figsize=(16, 14))
fig.suptitle('Sales Performance Analysis — POC Dashboard',
             fontsize=18, fontweight='bold', y=0.98)

# ── Chart 1: Profit by Region ─────────────────────────────
ax1 = fig.add_subplot(3, 3, 1)
rp = df.groupby('Region')['Profit'].sum().sort_values(ascending=False)
colors1 = ['#1D9E75' if v > 0 else '#E24B4A' for v in rp.values]
bars = ax1.bar(rp.index, rp.values, color=colors1, edgecolor='white', linewidth=0.5)
ax1.set_title('Profit by Region', fontweight='bold', fontsize=11)
ax1.set_ylabel('Profit ($)')
ax1.yaxis.set_major_formatter(mtick.FuncFormatter(lambda x, _: f'${x:,.0f}'))
for bar in bars:
    ax1.text(bar.get_x() + bar.get_width() / 2,
             bar.get_height() + 200,
             f'${bar.get_height():,.0f}',
             ha='center', va='bottom', fontsize=8)
ax1.tick_params(axis='x', labelsize=9)

# ── Chart 2: Sales by Category ────────────────────────────
ax2 = fig.add_subplot(3, 3, 2)
cs = df.groupby('Category')['Sales'].sum().sort_values()
ax2.barh(cs.index, cs.values,
         color=[PALETTE[c] for c in cs.index], edgecolor='white')
ax2.set_title('Sales by Category', fontweight='bold', fontsize=11)
ax2.set_xlabel('Total Sales ($)')
ax2.xaxis.set_major_formatter(mtick.FuncFormatter(lambda x, _: f'${x/1000:.0f}K'))
for i, v in enumerate(cs.values):
    ax2.text(v + 500, i, f'${v:,.0f}', va='center', fontsize=8)
ax2.tick_params(axis='y', labelsize=9)

# ── Chart 3: Monthly Sales Trend ──────────────────────────
ax3 = fig.add_subplot(3, 3, 3)
ms = df.groupby('Month Name')['Sales'].sum().reindex(MONTH_ORDER)
ax3.plot(ms.index, ms.values, marker='o', linewidth=2,
         color='#378ADD', markersize=5)
ax3.fill_between(range(len(ms)), ms.values, alpha=0.12, color='#378ADD')
ax3.set_xticks(range(len(ms)))
ax3.set_xticklabels(ms.index, rotation=45, fontsize=8)
ax3.set_title('Monthly Sales Trend', fontweight='bold', fontsize=11)
ax3.set_ylabel('Sales ($)')
ax3.yaxis.set_major_formatter(mtick.FuncFormatter(lambda x, _: f'${x/1000:.0f}K'))
peak = ms.idxmax()
pi = list(ms.index).index(peak)
ax3.annotate(f'Peak\n{peak}',
             xy=(pi, ms.max()),
             xytext=(pi - 1.5, ms.max() * 1.05),
             arrowprops=dict(arrowstyle='->', color='red', lw=1),
             color='red', fontsize=8)

# ── Chart 4: Sub-Category Sales ───────────────────────────
ax4 = fig.add_subplot(3, 3, 4)
sc = (df.groupby(['Category', 'Sub-Category'])['Sales']
        .sum().reset_index().sort_values('Sales'))
ax4.barh(sc['Sub-Category'],
         sc['Sales'],
         color=[PALETTE[c] for c in sc['Category']],
         alpha=0.85, edgecolor='white')
ax4.set_title('Sales by Sub-Category', fontweight='bold', fontsize=11)
ax4.set_xlabel('Sales ($)')
ax4.xaxis.set_major_formatter(mtick.FuncFormatter(lambda x, _: f'${x/1000:.0f}K'))
ax4.tick_params(axis='y', labelsize=9)
from matplotlib.patches import Patch
legend_elements = [Patch(facecolor=v, label=k, alpha=0.85)
                   for k, v in PALETTE.items()]
ax4.legend(handles=legend_elements, fontsize=7, loc='lower right')

# ── Chart 5: Top 10 Customers ─────────────────────────────
ax5 = fig.add_subplot(3, 3, 5)
tc = df.groupby('Customer Name')['Sales'].sum().nlargest(10).sort_values()
ax5.barh(tc.index, tc.values, color='#534AB7', alpha=0.85, edgecolor='white')
ax5.set_title('Top 10 Customers by Revenue', fontweight='bold', fontsize=11)
ax5.set_xlabel('Total Sales ($)')
ax5.xaxis.set_major_formatter(mtick.FuncFormatter(lambda x, _: f'${x:,.0f}'))
for i, v in enumerate(tc.values):
    ax5.text(v + 100, i, f'${v:,.0f}', va='center', fontsize=7)
ax5.tick_params(axis='y', labelsize=8)

# ── Chart 6: Sales vs Profit Scatter ──────────────────────
ax6 = fig.add_subplot(3, 3, 6)
for cat, color in PALETTE.items():
    sub = df[df['Category'] == cat]
    ax6.scatter(sub['Sales'], sub['Profit'],
                alpha=0.35, s=15, label=cat, color=color)
ax6.axhline(0, color='red', linewidth=0.8, linestyle='--', alpha=0.6)
ax6.set_title('Sales vs Profit', fontweight='bold', fontsize=11)
ax6.set_xlabel('Sales ($)')
ax6.set_ylabel('Profit ($)')
ax6.legend(fontsize=7)

# ── Chart 7: Discount vs Avg Profit ───────────────────────
ax7 = fig.add_subplot(3, 3, 7)
df['Disc Bucket'] = pd.cut(
    df['Discount'],
    bins=[-0.01, 0, 0.2, 0.4, 0.6],
    labels=['0%', '1-20%', '21-40%', '41-60%']
)
dp = df.groupby('Disc Bucket', observed=True)['Profit'].mean()
bar_c = ['#1D9E75' if v >= 0 else '#E24B4A' for v in dp.values]
ax7.bar(dp.index, dp.values, color=bar_c, edgecolor='white')
ax7.axhline(0, color='black', linewidth=0.8, linestyle='--', alpha=0.5)
ax7.set_title('Avg Profit by Discount', fontweight='bold', fontsize=11)
ax7.set_xlabel('Discount Range')
ax7.set_ylabel('Avg Profit ($)')
ax7.tick_params(axis='x', labelsize=9)

# ── Chart 8: Region × Category Heatmap ───────────────────
ax8 = fig.add_subplot(3, 3, 8)
pivot = df.pivot_table(
    values='Sales', index='Region',
    columns='Category', aggfunc='sum'
)
sns.heatmap(pivot, ax=ax8, annot=True, fmt='.0f',
            cmap='YlGn', linewidths=0.5,
            cbar_kws={'shrink': 0.8})
ax8.set_title('Sales Heatmap: Region × Category',
              fontweight='bold', fontsize=11)
ax8.tick_params(axis='x', labelsize=8, rotation=15)
ax8.tick_params(axis='y', labelsize=8, rotation=0)

# ── Chart 9: Correlation Matrix ───────────────────────────
ax9 = fig.add_subplot(3, 3, 9)
corr = df[['Sales', 'Profit', 'Quantity', 'Discount']].corr()
mask = np.triu(np.ones_like(corr, dtype=bool))
sns.heatmap(corr, ax=ax9, annot=True, fmt='.2f',
            cmap='RdYlGn', vmin=-1, vmax=1,
            linewidths=0.5, mask=mask,
            cbar_kws={'shrink': 0.8})
ax9.set_title('Correlation Matrix', fontweight='bold', fontsize=11)
ax9.tick_params(labelsize=8)

# ── Save & Show ───────────────────────────────────────────
plt.tight_layout(rect=[0, 0, 1, 0.97])
plt.savefig('sales_dashboard.png', dpi=130,
            bbox_inches='tight', facecolor='white')
plt.show()
print("\nDone! Dashboard saved as: sales_dashboard.png")

# ── 5. KEY INSIGHTS ───────────────────────────────────────
print("\n" + "=" * 50)
print("KEY INSIGHTS")
print("=" * 50)
print(f"Best profit region  : {rp.idxmax()} (${rp.max():,.0f})")
print(f"Top sales category  : {cs.idxmax()} (${cs.max():,.0f})")
print(f"Peak sales month    : {peak}")
print(f"Top customer        : {tc.idxmax()} (${tc.max():,.0f})")
print(f"Total revenue       : ${df['Sales'].sum():,.0f}")
print(f"Total profit        : ${df['Profit'].sum():,.0f}")
print(f"Overall margin      : {df['Profit'].sum()/df['Sales'].sum()*100:.1f}%")
print("=" * 50)
