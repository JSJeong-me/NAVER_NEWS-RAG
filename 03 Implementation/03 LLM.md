Where LLM is Used
The system uses LLM for:

Article Summarization (
llm/summarizer.py
) - Extract sentiment, impact, risk flags
Article Classification (
llm/classifier.py
) - Classify trade actions and regime
Optional features in recommendation engine and regime classifier (currently disabled by default)
Note: The current implementation already works without LLM for basic analysis (using rule-based Korean keyword matching). LLM is only needed for advanced features.

Need help implementing environment variable configuration or want to add support for other LLMs (Anthropic, local models, etc.)?

Great question! Here are several ways to set your OpenAI API key:

Method 1: Environment Variable (Recommended)
Temporary (Current Session Only)
bash
export OPENAI_API_KEY="sk-your-api-key-here"
Permanent (Add to ~/.bashrc)
bash
echo 'export OPENAI_API_KEY="sk-your-api-key-here"' >> ~/.bashrc
source ~/.bashrc
Method 2: .env File (For Development)
Create a .env file in the project root:

bash
cd /home/ubuntu/news_regime
cat > .env << 'EOF'
OPENAI_API_KEY=sk-your-api-key-here
EOF
Then install and use python-dotenv:

bash
pip install python-dotenv
Update 
config/loader.py
 to load it:

python
from dotenv import load_dotenv
import os

class ConfigLoader:
    def __init__(self, env: Optional[str] = None):
        load_dotenv()  # Load .env file
        self.env = env or "dev"
Method 3: Direct in Code (Not Recommended - For Testing Only)
Modify 
llm/client.py
:

python
def __init__(self, config: LlmClientConfig) -> None:
    self.config = config
    self._client = OpenAI(
        api_key="sk-your-api-key-here",  # Hardcoded (not recommended)
        timeout=config.request_timeout
    )
Verify It's Set
bash
# Check if environment variable is set
echo $OPENAI_API_KEY

# Test with Python
python -c "import os; print('API Key:', os.getenv('OPENAI_API_KEY', 'NOT SET')[:20] + '...')"
Current Implementation
The OpenAI client in 
llm/client.py
 automatically reads from the OPENAI_API_KEY environment variable (no code changes needed):

python
self._client = OpenAI(timeout=config.request_timeout)
# ↑ Automatically uses OPENAI_API_KEY from environment
Get Your API Key
If you don't have an API key yet:

Go to https://platform.openai.com/api-keys
Click "Create new secret key"
Copy the key (starts with sk-)
Set it using one of the methods above
Security Note
⚠️ Never commit API keys to Git! Add .env to .gitignore:

bash
echo ".env" >> /home/ubuntu/news_regime/.gitignore
Recommended approach: Use Method 1 (environment variable in ~/.bashrc) for development, or Method 2 (.env file) if you prefer keeping secrets in a file.
