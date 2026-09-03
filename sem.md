 ### How it works                                                           
                                                                             
  The semantic search relies on two main components: AI Embeddings and       
  PostgreSQL's pgvector extension for vector similarity search.              
                                                                             
  1. Generating Embeddings: Whenever a new entity (like a Cost Center) is    
  created or an item (like an AP Bill Item) needs to be matched, the         
  application concatenates relevant fields (e.g., name, description,         
  category) and sends this text to an AI Model (like OpenAI or an internal   
  model) using the GenerateEmbeddings function to get a numerical vector     
  representation (e.g., of dimension 1536).                                  
  2. Storing the Vectors: The generated embeddings are cast into a pgvector  
  format using pgvector.NewVector(embeddings) and stored natively in a       
  PostgreSQL table under an embeddings column.                               
  3. Semantic Matching (Vector Search): When searching (for example, finding 
  the closest Chart of Account for a bill item), the query text is similarly 
  converted into an embedding. The backend then runs a raw SQL query using   
  pgvector's cosine distance operator (<=>) to find the nearest neighbors in 
  the database:                                                              
    SELECT * FROM <table_name> WHERE domain = '<domain>' ORDER BY embeddings 
  <=> ? LIMIT ?                                                              
                                                                             
                                                                             
  ### Where to start your deep dive                                          
                                                                             
  If you want to read through the code to see this in action, I recommend    
  exploring these specific files:                                            
                                                                             
  1. Example 1: Creating and Storing Embeddings (Cost Center)                
                                                                             
  • File: costcenter.go:650-670                                              
  • What to look for: Look around line 665. You'll see GenerateEmbeddings    
  being called with the Cost Center's concatenated text (InputText:          
  subEntity.Name + " " + subEntity.Description), followed by pgvector.       
  NewVector(embeddings) being attached to the CostCenter struct and saved to 
  the database.                                                              
                                                                             
  2. Example 2: Searching via Semantic Text (Chart of Accounts)              
                                                                             
  • Service Layer: chartofaccounts.go:252-280                                
  Look at the DetectChartOfAccountBySemanticText function. It takes the text 
  query, generates an embedding, and calls the data store layer.             
  • Database Layer: chartofaccounts.go:149-172                               
  Look at the SearchChartOfAccountVectors function. This is where the raw SQL
  using the pgvector cosine distance operator (ORDER BY embeddings <=> ?     
  LIMIT ?) is executed against PostgreSQL.
  
  3. The Dedicated Semantic Engine Agent (Optional)
  
  • File: engine.go
  • What to look for: If you are interested in how the generalized VapusData 
  semantic engine works outside of just the finance app, this file contains  
  the agent responsible for chunking documents (like PDFs/OCR results),      
  indexing them, and building searchable semantic indices.
