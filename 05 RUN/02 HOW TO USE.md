🚀 How to Use
Quick commands:



cd /home/ubuntu/news_regime

# See everything
./query_db.sh all

# Just news
./list_news.sh

# Specific queries
./query_db.sh analysis
./query_db.sh recommendations
./query_db.sh regime


With options:


./query_db.sh news --hours 48 --limit 10
./query_db.sh all --hours 24 --limit 5


📁 New Files Created
/home/ubuntu/news_regime/cli/query_db.py
 - Query tool implementation
/home/ubuntu/news_regime/query_db.sh
 - Wrapper script
/home/ubuntu/news_regime/list_news.sh
 - News listing script
/home/ubuntu/news_regime/SQL_DOCUMENTATION.md
 - Complete SQL docs
/home/ubuntu/news_regime/SQL_IMPLEMENTATION_SUMMARY.md
 - Summary
/home/ubuntu/news_regime/QUICK_REFERENCE.md


 - Quick reference
✨ Key Features
Multiple query modes - News, analysis, symbols, recommendations, regime
Flexible filtering - Time windows and result limits
Statistical summaries - Average sentiment, impact scores, risk counts
Grouped views - Symbols grouped by article count
Clean output - Formatted with separators and clear sections
Easy to use - Simple wrapper scripts, no Python knowledge needed
All SQL work is complete and fully functional! The tools are ready to use. 🎉
