# 1. System Overview

The AI-Powered Personalized E-Commerce Decision Engine implements a serverless-first, microservices architecture on AWS to deliver sub-3-second personalized shopping recommendations. The system is an **in-app AI-powered decision engine** that dynamically analyzes user visual inputs and grounds recommendations using GPT-4o + RAG-based retail knowledge.

## Solution Approach

Instead of forcing customers to guess, the system recommends the highest-confidence purchase option by eliminating low-confidence matches in real time:

- **📸 Scan your foot** → Estimates size range & fit probability using visual cues + brand-specific sizing knowledge
- **🤳 Scan your face** → Infers skin profile and filters ingredient-compatible products
- **👕 Scan your body** → Suggests high-confidence fit options using structured apparel data

The system processes biometric data through camera-based scanning, extracts structured attributes without persistent biometric storage, and leverages Amazon Bedrock with Retrieval Augmented Generation (RAG) to provide high-confidence purchase decisions.

## Core Design Philosophy (Decision Personalization)

The architecture prioritizes **decision confidence over choice abundance**. Rather than overwhelming users with product options, the system dynamically analyzes visual inputs and applies intelligent filtering to present only high-confidence matches (>75% confidence threshold) with explainable reasoning. This approach eliminates the guesswork, reduces cognitive load, and addresses the primary cause of cart abandonment in Indian e-commerce markets.

## Privacy-First and In-Memory Biometric Processing Model

All biometric data processing occurs in ephemeral compute environments with automatic memory cleanup. Raw images are processed in-memory buffers with maximum 30-second lifecycle, immediately purged post-analysis. Only structured, non-identifiable attributes are persisted in encrypted storage, ensuring compliance with Indian Personal Data Protection Act and GDPR requirements.

## Indian Retail Constraints Consideration

The architecture accommodates Tier 2/3 connectivity limitations through Progressive Web App (PWA) capabilities, offline-first design patterns, and intelligent caching strategies. Regional sizing variations, climate-based product suitability, and cultural preferences are embedded into the knowledge base and recommendation algorithms.

# 2. Architectural Principles

## Scalability
- Horizontal auto-scaling across all compute layers
- Stateless service design enabling elastic scaling
- Database sharding strategies for user and product data
- CDN-based content distribution for global performance

## Security by Design
- Zero-trust network architecture with IAM role isolation
- Encryption at rest and in transit using AWS KMS
- API Gateway throttling and DDoS protection
- Regular automated security scanning and compliance validation

## Privacy by Design
- Data minimization with structured attribute extraction only
- Consent-driven data collection with granular permissions
- Automated data retention and deletion workflows
- Anonymization pipelines for machine learning training

## Low-Latency Inference
- Edge computing for image preprocessing
- Optimized vector similarity search with OpenSearch
- Connection pooling and persistent connections
- Intelligent caching with Redis for frequent queries

## Modular Microservices
- Domain-driven service boundaries
- Async communication patterns with event-driven architecture
- Independent deployment and scaling capabilities
- Service mesh for inter-service communication and observability

## Observability-First
- Structured logging with correlation IDs
- Distributed tracing across service boundaries
- Real-time metrics and alerting with CloudWatch
- Business intelligence dashboards for conversion tracking

# 3. High-Level Architecture Diagram Description

```
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 1: Client Layer                        │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │   Web Browser   │    │   Mobile PWA    │                    │
│  │   (React SPA)   │    │   (React PWA)   │                    │
│  └─────────────────┘    └─────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              Layer 2: CDN & Frontend Hosting                    │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │   CloudFront    │    │   AWS Amplify   │                    │
│  │   (Global CDN)  │    │   (Hosting)     │                    │
│  └─────────────────┘    └─────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                Layer 3: Authentication                          │
│                ┌─────────────────┐                             │
│                │  Amazon Cognito │                             │
│                │  (User Pools)   │                             │
│                └─────────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 4: API Layer                          │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │   API Gateway   │    │   FastAPI on    │                    │
│  │   (REST + WS)   │    │   EC2 Cluster   │                    │
│  └─────────────────┘    └─────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                Layer 5: AI Inference Layer                     │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │ Amazon Bedrock  │    │  Knowledge Base │                    │
│  │   (GPT-4o)      │    │   (RAG Store)   │                    │
│  └─────────────────┘    └─────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 6: Data Layer                         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │  DynamoDB   │ │     S3      │ │ OpenSearch  │              │
│  │ (NoSQL DB)  │ │ (Storage)   │ │ (Vector DB) │              │
│  └─────────────┘ └─────────────┘ └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Layer 7: Monitoring                          │
│                ┌─────────────────┐                             │
│                │  CloudWatch +   │                             │
│                │  X-Ray Tracing  │                             │
│                └─────────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

## Component Interactions

1. **Client → CDN**: Static assets served via CloudFront with edge caching
2. **Client → Cognito**: JWT token acquisition and refresh workflows
3. **Client → API Gateway**: Authenticated API calls with rate limiting
4. **API Gateway → FastAPI**: Request routing and load balancing
5. **FastAPI → Bedrock**: AI inference requests with structured prompts
6. **FastAPI → OpenSearch**: Vector similarity search for product matching
7. **FastAPI → DynamoDB**: User profile and product metadata operations
8. **All Services → CloudWatch**: Centralized logging and metrics collection

# 4. Detailed Component Design

## 4.1 Frontend Design

### React Architecture
```
src/
├── components/
│   ├── camera/
│   │   ├── CameraCapture.tsx
│   │   ├── ScanProgress.tsx
│   │   └── ScanResults.tsx
│   ├── recommendations/
│   │   ├── ProductCard.tsx
│   │   ├── ConfidenceIndicator.tsx
│   │   └── RecommendationList.tsx
│   └── shared/
│       ├── LoadingSpinner.tsx
│       └── ErrorBoundary.tsx
├── hooks/
│   ├── useCamera.ts
│   ├── useAuth.ts
│   └── useRecommendations.ts
├── services/
│   ├── api.ts
│   ├── auth.ts
│   └── camera.ts
└── store/
    ├── authSlice.ts
    ├── scanSlice.ts
    └── recommendationSlice.ts
```

### Camera Integration Flow
1. **Permission Request**: Navigator.mediaDevices.getUserMedia() with fallback handling
2. **Stream Initialization**: Video stream setup with optimal resolution (1920x1080)
3. **Capture Process**: Canvas-based image capture with quality validation
4. **Upload Preparation**: Image compression and format standardization (JPEG, <2MB)
5. **Progress Tracking**: Real-time upload progress with retry mechanisms

### State Management
- **Redux Toolkit** for global state management
- **RTK Query** for API state synchronization
- **Persistent storage** using Redux Persist for offline capabilities
- **Optimistic updates** for improved user experience

### Secure Token Handling
- **JWT storage** in httpOnly cookies with SameSite=Strict
- **Automatic refresh** using refresh token rotation
- **Token expiration** handling with seamless re-authentication
- **Logout cleanup** with token revocation

### Confidence Score Visualization
```typescript
interface ConfidenceIndicator {
  score: number; // 0-100
  factors: {
    fit_accuracy: number;
    size_match: number;
    style_preference: number;
    availability: number;
  };
  explanation: string;
}
```

## 4.2 API Gateway Design

### Route Structure
```
/api/v1/
├── /auth/
│   ├── POST /login
│   ├── POST /refresh
│   └── POST /logout
├── /scan/
│   ├── POST /foot
│   ├── POST /face
│   └── POST /body
├── /recommendations/
│   ├── GET /products
│   ├── GET /confidence/{product_id}
│   └── POST /feedback
└── /user/
    ├── GET /profile
    ├── PUT /preferences
    └── DELETE /account
```

### Rate Limiting
- **Tier-based limits**: 100 req/min for authenticated users, 10 req/min for anonymous
- **Burst capacity**: 200 requests with token bucket algorithm
- **Scan-specific limits**: 5 scans per hour per user to prevent abuse
- **Geographic throttling**: Enhanced limits for Tier 2/3 city IP ranges

### JWT Validation
- **Cognito integration** with automatic token verification
- **Scope-based authorization** for different API endpoints
- **Custom claims validation** for user roles and permissions
- **Token blacklisting** for immediate revocation support

### Error Handling Strategy
```json
{
  "error": {
    "code": "SCAN_PROCESSING_FAILED",
    "message": "Unable to extract attributes from image",
    "details": {
      "retry_after": 30,
      "suggestions": ["Ensure good lighting", "Hold camera steady"]
    },
    "correlation_id": "req_123456789"
  }
}
```

## 4.3 Backend (FastAPI on EC2)

### Service Modularization
```
app/
├── services/
│   ├── scan_service.py
│   ├── attribute_extraction_service.py
│   ├── recommendation_service.py
│   └── feedback_service.py
├── models/
│   ├── user_models.py
│   ├── product_models.py
│   └── scan_models.py
├── api/
│   ├── scan_router.py
│   ├── recommendation_router.py
│   └── feedback_router.py
└── core/
    ├── config.py
    ├── dependencies.py
    └── security.py
```

### Scan Service
```python
class ScanService:
    """
    Dynamically analyzes user visual inputs in real-time.
    Eliminates low-confidence matches to provide highest-confidence purchase options.
    """
    
    async def process_foot_scan(
        self, 
        image: UploadFile, 
        user_id: str
    ) -> FootAttributes:
        """
        📸 Scan your foot → Estimates size range & fit probability 
        using visual cues + brand-specific sizing knowledge
        """
        # In-memory processing with automatic cleanup
        image_buffer = await self._load_to_memory(image)
        try:
            # Extract visual cues from foot image
            attributes = await self._extract_foot_attributes(image_buffer)
            
            # Ground with brand-specific sizing knowledge via RAG
            brand_sizing_context = await self._retrieve_brand_sizing_knowledge()
            
            # Calculate fit probability for each brand
            attributes.fit_probabilities = await self._calculate_brand_fit_probabilities(
                attributes, brand_sizing_context
            )
            
            return await self._validate_and_structure(attributes)
        finally:
            del image_buffer  # Explicit memory cleanup
    
    async def process_face_scan(
        self, 
        image: UploadFile, 
        user_id: str
    ) -> FaceAttributes:
        """
        🤳 Scan your face → Infers skin profile and filters 
        ingredient-compatible products
        """
        image_buffer = await self._load_to_memory(image)
        try:
            # Infer skin profile from visual analysis
            skin_profile = await self._infer_skin_profile(image_buffer)
            
            # Retrieve ingredient compatibility knowledge
            ingredient_knowledge = await self._retrieve_ingredient_knowledge()
            
            # Filter compatible products dynamically
            skin_profile.compatible_ingredients = await self._filter_compatible_ingredients(
                skin_profile, ingredient_knowledge
            )
            
            return await self._validate_and_structure(skin_profile)
        finally:
            del image_buffer
    
    async def process_body_scan(
        self, 
        image: UploadFile, 
        user_id: str
    ) -> BodyAttributes:
        """
        👕 Scan your body → Suggests high-confidence fit options 
        using structured apparel data
        """
        image_buffer = await self._load_to_memory(image)
        try:
            # Extract body measurements from visual input
            measurements = await self._extract_body_measurements(image_buffer)
            
            # Retrieve structured apparel data via RAG
            apparel_data = await self._retrieve_apparel_data()
            
            # Calculate high-confidence fit options
            measurements.fit_options = await self._calculate_high_confidence_fits(
                measurements, apparel_data
            )
            
            return await self._validate_and_structure(measurements)
        finally:
            del image_buffer
```

### Attribute Extraction Service
- **Computer vision models** for biometric analysis
- **Structured output validation** using Pydantic models
- **Confidence scoring** for each extracted attribute
- **Error handling** for poor quality images

### Recommendation Service
- **Dynamic visual input analysis** with real-time attribute extraction
- **Vector similarity search** integration with OpenSearch
- **RAG query construction** for Bedrock inference with retail knowledge grounding
- **Confidence threshold filtering** (>75% minimum) to eliminate low-confidence matches
- **Real-time personalization** based on visual inputs and user history
- **Highest-confidence option selection** instead of overwhelming product lists

### Feedback Service
- **Async feedback processing** to prevent blocking
- **Structured feedback categorization** for model improvement
- **Batch processing** for model retraining triggers
- **Analytics integration** for business intelligence

### Dependency Injection Pattern
```python
@lru_cache()
def get_settings():
    return Settings()

def get_database():
    return DynamoDBClient(get_settings().aws_region)

def get_bedrock_client():
    return BedrockClient(get_settings().bedrock_model_id)
```

### Async Processing Model
- **AsyncIO** for concurrent request handling
- **Background tasks** for non-blocking operations
- **Connection pooling** for database and external services
- **Circuit breakers** for external service failures

### Timeout Handling
- **Request timeouts**: 30 seconds for scan processing
- **Database timeouts**: 5 seconds for DynamoDB operations
- **AI inference timeouts**: 10 seconds for Bedrock calls
- **Graceful degradation** with fallback responses

## 4.4 AI Layer Design

### Attribute Extraction

#### Computer Vision Processing Flow
```python
class AttributeExtractor:
    """
    Dynamically analyzes visual inputs to extract structured attributes.
    Grounds analysis with retail knowledge for high-confidence recommendations.
    """
    
    async def extract_foot_attributes(self, image_buffer: bytes) -> FootAttributes:
        """
        📸 Foot Scan Processing:
        Estimates size range & fit probability using visual cues + brand-specific sizing knowledge
        """
        # 1. Image preprocessing
        processed_image = await self._preprocess_image(image_buffer)
        
        # 2. Visual cue detection (keypoints, contours, dimensions)
        keypoints = await self._detect_foot_keypoints(processed_image)
        
        # 3. Measurement calculation from visual cues
        measurements = await self._calculate_measurements(keypoints)
        
        # 4. Size estimation with brand-specific knowledge
        size_range = await self._estimate_size_range(measurements)
        
        # 5. Fit probability calculation per brand
        fit_probabilities = await self._calculate_brand_fit_probabilities(
            measurements, size_range
        )
        
        return FootAttributes(
            length_mm=measurements.length,
            width_mm=measurements.width,
            arch_type=measurements.arch_type,
            size_range=size_range,
            fit_probabilities=fit_probabilities,  # Per-brand confidence scores
            confidence=measurements.confidence
        )
    
    async def extract_face_attributes(self, image_buffer: bytes) -> FaceAttributes:
        """
        🤳 Face Scan Processing:
        Infers skin profile and filters ingredient-compatible products
        """
        # 1. Skin analysis from visual input
        skin_analysis = await self._analyze_skin_from_image(image_buffer)
        
        # 2. Skin profile inference
        skin_profile = await self._infer_skin_profile(skin_analysis)
        
        # 3. Ingredient compatibility filtering
        compatible_ingredients = await self._filter_compatible_ingredients(skin_profile)
        incompatible_ingredients = await self._identify_incompatible_ingredients(skin_profile)
        
        return FaceAttributes(
            skin_tone=skin_profile.tone,
            skin_type=skin_profile.type,
            sensitivity_level=skin_profile.sensitivity,
            compatible_ingredients=compatible_ingredients,
            incompatible_ingredients=incompatible_ingredients,
            age_range=skin_profile.age_range,
            climate_suitability=skin_profile.climate_preferences,
            confidence=skin_analysis.confidence
        )
    
    async def extract_body_attributes(self, image_buffer: bytes) -> BodyAttributes:
        """
        👕 Body Scan Processing:
        Suggests high-confidence fit options using structured apparel data
        """
        # 1. Body measurement extraction from visual input
        body_analysis = await self._analyze_body_from_image(image_buffer)
        
        # 2. Structured measurement calculation
        measurements = await self._calculate_body_measurements(body_analysis)
        
        # 3. High-confidence fit option identification
        fit_options = await self._identify_high_confidence_fits(measurements)
        
        # 4. Eliminate low-confidence matches
        filtered_options = await self._eliminate_low_confidence_matches(
            fit_options, confidence_threshold=0.75
        )
        
        return BodyAttributes(
            measurements=measurements.structured_data,
            body_type=measurements.body_type,
            fit_preferences=measurements.fit_preferences,
            high_confidence_fits=filtered_options,  # Only >75% confidence
            confidence=body_analysis.confidence
        )
```

#### Structured Output Schema
```python
class FootAttributes(BaseModel):
    """
    📸 Foot scan output: Size range & fit probability with brand-specific knowledge
    """
    length_mm: float
    width_mm: float
    arch_type: Literal["low", "normal", "high"]
    size_range: Dict[str, Union[int, float]]
    fit_probabilities: Dict[str, float]  # Brand → fit probability mapping
    brand_recommendations: List[BrandSizeRecommendation]
    confidence: float
    extracted_at: datetime

class FaceAttributes(BaseModel):
    """
    🤳 Face scan output: Skin profile with ingredient compatibility filtering
    """
    skin_tone: str
    skin_type: Literal["oily", "dry", "combination", "sensitive"]
    sensitivity_level: Literal["low", "medium", "high"]
    compatible_ingredients: List[str]  # Filtered compatible ingredients
    incompatible_ingredients: List[str]  # Ingredients to avoid
    age_range: Tuple[int, int]
    climate_suitability: List[str]
    product_filters: Dict[str, Any]  # Dynamic filters for product matching
    confidence: float
    extracted_at: datetime

class BodyAttributes(BaseModel):
    """
    👕 Body scan output: High-confidence fit options with structured apparel data
    """
    measurements: Dict[str, float]
    body_type: str
    fit_preferences: List[str]
    high_confidence_fits: List[FitOption]  # Only >75% confidence matches
    size_recommendations: Dict[str, str]  # Category → recommended size
    confidence: float
    extracted_at: datetime

class FitOption(BaseModel):
    """
    High-confidence fit option with elimination of low-confidence matches
    """
    product_category: str
    recommended_size: str
    fit_confidence: float  # Must be >75%
    fit_type: Literal["slim", "regular", "relaxed"]
    reasoning: str  # Explainable AI reasoning
    brand_specific_notes: Optional[str]
```

#### Temporary In-Memory Image Buffer
- **Maximum buffer size**: 10MB per image
- **Automatic cleanup**: 30-second TTL with explicit deletion
- **Memory monitoring**: Real-time memory usage tracking
- **Overflow protection**: Request rejection when memory threshold exceeded

### Vectorization

#### Embedding Model Usage
```python
class EmbeddingService:
    def __init__(self):
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
    
    async def create_user_embedding(
        self, 
        attributes: UserAttributes
    ) -> np.ndarray:
        # Combine all user attributes into structured text
        attribute_text = self._serialize_attributes(attributes)
        
        # Generate 512-dimensional embedding
        embedding = self.model.encode(attribute_text)
        
        return embedding.astype(np.float32)
```

#### 512-Dimension Embedding Creation
- **Attribute serialization** into structured text format
- **Normalization** for consistent vector magnitudes
- **Dimensionality reduction** using PCA if needed
- **Quality validation** with embedding similarity checks

#### OpenSearch Vector Indexing
```json
{
  "mappings": {
    "properties": {
      "user_id": {"type": "keyword"},
      "embedding_vector": {
        "type": "dense_vector",
        "dims": 512,
        "index": true,
        "similarity": "cosine"
      },
      "attributes": {"type": "object"},
      "created_at": {"type": "date"}
    }
  }
}
```

### RAG Flow

#### Query Formulation
```python
class RAGQueryBuilder:
    """
    Builds queries that ground recommendations in retail knowledge.
    Dynamically analyzes visual inputs to eliminate low-confidence matches.
    """
    
    def build_recommendation_query(
        self, 
        user_attributes: UserAttributes,
        product_category: str
    ) -> str:
        return f"""
        User Profile (from visual input analysis):
        - Foot: Size range {user_attributes.foot.size_range}, Fit probabilities: {user_attributes.foot.fit_probabilities}
        - Skin: Type {user_attributes.face.skin_type}, Compatible ingredients: {user_attributes.face.compatible_ingredients}
        - Body: Measurements {user_attributes.body.measurements}, High-confidence fits: {user_attributes.body.high_confidence_fits}
        - Climate: {user_attributes.location.climate}
        
        Task: Find products in {product_category} with HIGHEST confidence match.
        
        Requirements:
        1. Use brand-specific sizing knowledge to calculate fit probability
        2. Filter products by ingredient compatibility (exclude incompatible ingredients)
        3. Match against structured apparel data for fit confidence
        4. Eliminate all matches with confidence < 75%
        5. Return ONLY the highest-confidence options (top 3-5)
        6. Provide clear reasoning for each recommendation
        
        Instead of showing all possible products, recommend the purchase option 
        the user can be most confident about.
        """
```

#### Knowledge Retrieval
- **Semantic search** using vector similarity in knowledge base for retail domain expertise
- **Brand-specific sizing knowledge** retrieval for accurate fit probability calculation
- **Ingredient compatibility database** for skincare product filtering
- **Structured apparel data** retrieval for high-confidence fit matching
- **Hybrid retrieval** combining keyword and vector search
- **Context ranking** based on relevance scores and confidence thresholds
- **Knowledge base versioning** for A/B testing and continuous improvement

#### Prompt Construction
```python
class PromptTemplate:
    """
    Constructs prompts that leverage GPT-4o + RAG for grounded recommendations.
    Emphasizes elimination of low-confidence matches and highest-confidence selection.
    """
    
    RECOMMENDATION_TEMPLATE = """
    You are an AI-powered decision engine for e-commerce. Your goal is to help users 
    make confident purchase decisions by analyzing their visual inputs and grounding 
    recommendations in retail knowledge.
    
    User Visual Input Analysis:
    - Foot Scan: {foot_attributes}
      → Size range & fit probability using brand-specific sizing knowledge
    - Face Scan: {face_attributes}
      → Skin profile with ingredient compatibility filtering
    - Body Scan: {body_attributes}
      → High-confidence fit options using structured apparel data
    
    Available Products (pre-filtered): {retrieved_products}
    Retail Knowledge Base: {domain_knowledge}
    
    Instructions:
    1. Calculate confidence score for each product using:
       - Fit probability from brand-specific sizing data
       - Ingredient compatibility from skin profile analysis
       - Fit confidence from structured apparel data matching
       - Product metadata quality and availability
    
    2. ELIMINATE all products with confidence < 75%
    
    3. Select the HIGHEST-CONFIDENCE options (3-5 products maximum)
    
    4. For each recommendation, provide:
       - Confidence score (0-100)
       - Fit probability with brand-specific reasoning
       - Ingredient compatibility explanation (for skincare/cosmetics)
       - Size recommendation with fit type
       - Clear reasoning why this is a high-confidence match
       - Any potential concerns or considerations
    
    5. Rank by confidence score (highest first)
    
    Remember: Instead of forcing customers to guess, recommend the purchase option 
    they can be most confident about. Quality over quantity.
    """
```

#### Confidence Scoring Computation
```python
def calculate_confidence_score(
    fit_score: float,
    attribute_similarity: float,
    product_metadata_quality: float,
    brand_reliability: float
) -> float:
    """
    Confidence = weighted_average([
        fit_score * 0.4,
        attribute_similarity * 0.3,
        product_metadata_quality * 0.2,
        brand_reliability * 0.1
    ])
    """
    weights = [0.4, 0.3, 0.2, 0.1]
    scores = [fit_score, attribute_similarity, product_metadata_quality, brand_reliability]
    
    confidence = sum(w * s for w, s in zip(weights, scores))
    return min(max(confidence, 0.0), 100.0)
```

#### Final Filtering Logic
- **Confidence threshold**: Minimum 75% confidence required (eliminates low-confidence matches)
- **Availability filtering**: Only in-stock products in user's region
- **Price range filtering**: Based on user's budget preferences
- **Ingredient exclusion**: Remove products with incompatible ingredients (from face scan)
- **Fit confidence filtering**: Only include high-confidence fit options (from body scan)
- **Brand fit probability**: Prioritize brands with highest fit probability (from foot scan)
- **Diversity enforcement**: Maximum 2 products per brand in results
- **Highest-confidence selection**: Rank by confidence score and return top 3-5 options only

The system dynamically eliminates low-confidence matches in real-time, ensuring users see only the purchase options they can be most confident about.

# 5. Data Design

## 5.1 DynamoDB Tables

### UserAttributes Table
```
Table: user-attributes
Partition Key: user_id (String)
Sort Key: attribute_type (String) # foot, face, body

Attributes:
- user_id: String
- attribute_type: String
- attributes: Map
- embedding_vector: List<Number>
- confidence_score: Number
- created_at: String (ISO 8601)
- updated_at: String (ISO 8601)
- ttl: Number (Unix timestamp)

GSI: user-created-index
- Partition Key: user_id
- Sort Key: created_at
```

### ProductMetadata Table
```
Table: product-metadata
Partition Key: product_id (String)

Attributes:
- product_id: String
- brand: String
- category: String
- subcategory: String
- sizing_data: Map
- ingredients: List<String>
- fit_data: Map
- region_availability: List<String>
- price_info: Map
- embedding_vector: List<Number>
- metadata_quality_score: Number
- created_at: String
- updated_at: String

GSI: brand-category-index
- Partition Key: brand
- Sort Key: category

GSI: category-price-index
- Partition Key: category
- Sort Key: price_info.min_price
```

### Feedback Table
```
Table: user-feedback
Partition Key: user_id (String)
Sort Key: feedback_id (String)

Attributes:
- user_id: String
- feedback_id: String
- product_id: String
- purchase_outcome: String # purchased, returned, cancelled
- satisfaction_rating: Number (1-5)
- fit_accuracy: Number (1-5)
- return_reason: String
- feedback_text: String
- created_at: String
- processed: Boolean

GSI: product-feedback-index
- Partition Key: product_id
- Sort Key: created_at
```

### ConsentLogs Table
```
Table: consent-logs
Partition Key: user_id (String)
Sort Key: consent_timestamp (String)

Attributes:
- user_id: String
- consent_timestamp: String
- consent_type: String # biometric, data_usage, marketing
- consent_granted: Boolean
- consent_version: String
- ip_address: String
- user_agent: String
- expiry_date: String
```

## 5.2 OpenSearch Design

### Vector Index Mapping
```json
{
  "settings": {
    "index": {
      "knn": true,
      "knn.algo_param.ef_search": 100
    }
  },
  "mappings": {
    "properties": {
      "user_id": {"type": "keyword"},
      "product_id": {"type": "keyword"},
      "vector_type": {"type": "keyword"},
      "embedding": {
        "type": "knn_vector",
        "dimension": 512,
        "method": {
          "name": "hnsw",
          "space_type": "cosinesimil",
          "engine": "nmslib"
        }
      },
      "metadata": {"type": "object"},
      "created_at": {"type": "date"}
    }
  }
}
```

### Embedding Similarity Search
```python
async def find_similar_products(
    user_embedding: List[float],
    category: str,
    limit: int = 20
) -> List[ProductMatch]:
    query = {
        "size": limit,
        "query": {
            "bool": {
                "must": [
                    {"term": {"category": category}},
                    {
                        "knn": {
                            "embedding": {
                                "vector": user_embedding,
                                "k": limit
                            }
                        }
                    }
                ]
            }
        }
    }
    
    response = await opensearch_client.search(
        index="product-embeddings",
        body=query
    )
    
    return [
        ProductMatch(
            product_id=hit["_source"]["product_id"],
            similarity_score=hit["_score"],
            metadata=hit["_source"]["metadata"]
        )
        for hit in response["hits"]["hits"]
    ]
```

### Hybrid Search (Keyword + Vector)
```json
{
  "query": {
    "hybrid": {
      "queries": [
        {
          "match": {
            "description": "summer dress cotton"
          }
        },
        {
          "knn": {
            "embedding": {
              "vector": [0.1, 0.2, ...],
              "k": 10
            }
          }
        }
      ]
    }
  }
}
```

## 5.3 S3 Usage

### Temporary Object Storage
- **Bucket**: `temp-scan-processing-{environment}`
- **Lifecycle**: 1-hour automatic deletion
- **Encryption**: SSE-KMS with customer-managed keys
- **Access**: Pre-signed URLs with 5-minute expiration

### Log Archive Bucket
- **Bucket**: `app-logs-archive-{environment}`
- **Lifecycle**: 
  - Standard → IA after 30 days
  - IA → Glacier after 90 days
  - Glacier → Deep Archive after 365 days
- **Partitioning**: `year/month/day/hour/` structure

### Encryption Configuration
```json
{
  "Rules": [
    {
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:region:account:key/key-id"
      },
      "BucketKeyEnabled": true
    }
  ]
}
```

# 6. Security Design

## IAM Role Separation
```
Roles:
├── ECommerceAPI-ExecutionRole
│   ├── DynamoDB: Read/Write to user-* tables
│   ├── OpenSearch: Query and index operations
│   ├── Bedrock: InvokeModel permissions
│   └── CloudWatch: Logs and metrics
├── ScanProcessor-Role
│   ├── S3: GetObject on temp bucket only
│   ├── Bedrock: InvokeModel for vision tasks
│   └── CloudWatch: Logs
└── FeedbackProcessor-Role
    ├── DynamoDB: Write to feedback table
    ├── S3: PutObject to analytics bucket
    └── CloudWatch: Logs
```

## API Gateway Throttling
- **Account-level**: 10,000 requests per second
- **Stage-level**: 5,000 requests per second
- **Method-level**: 
  - Scan endpoints: 100 requests per second
  - Recommendation endpoints: 500 requests per second
- **Usage plans**: Tiered limits based on user subscription

## KMS Encryption
```
Key Policies:
├── Application-Data-Key
│   ├── Encrypt/Decrypt: API services only
│   ├── GenerateDataKey: For S3 and DynamoDB
│   └── Audit: CloudTrail logging enabled
└── User-Biometric-Key
    ├── Encrypt/Decrypt: Scan processor only
    ├── Short-term: 1-hour key rotation
    └── Audit: Enhanced logging
```

## TLS Enforcement
- **Minimum version**: TLS 1.2
- **Cipher suites**: ECDHE-RSA-AES256-GCM-SHA384 and stronger
- **Certificate management**: AWS Certificate Manager with auto-renewal
- **HSTS headers**: max-age=31536000; includeSubDomains

## Cognito User Pools
```
Configuration:
├── Password Policy:
│   ├── Minimum length: 12 characters
│   ├── Require: uppercase, lowercase, numbers, symbols
│   └── Temporary password: 24-hour expiration
├── MFA:
│   ├── SMS backup
│   ├── TOTP preferred
│   └── Recovery codes
└── Advanced Security:
    ├── Adaptive authentication
    ├── Compromised credentials detection
    └── Risk-based authentication
```

## Data Anonymization for ML Training
```python
class DataAnonymizer:
    def anonymize_user_data(self, user_data: UserData) -> AnonymizedData:
        return AnonymizedData(
            user_id=self._hash_user_id(user_data.user_id),
            attributes=user_data.attributes,
            demographic_cluster=self._get_cluster_id(user_data),
            timestamp=user_data.created_at.replace(
                minute=0, second=0, microsecond=0
            )
        )
```

## Right-to-Delete Workflow
1. **User request**: Authenticated deletion request via API
2. **Verification**: Multi-factor authentication required
3. **Data identification**: Scan all tables for user references
4. **Cascade deletion**: Remove user data from all systems
5. **Confirmation**: Email confirmation with deletion certificate
6. **Audit trail**: Immutable deletion log for compliance

# 7. Performance & Scalability Design

## Auto Scaling Groups
```
Configuration:
├── API-Servers-ASG:
│   ├── Min: 2 instances
│   ├── Max: 20 instances
│   ├── Target: CPU 70%, Memory 80%
│   └── Scale-out: +2 instances, 2-minute cooldown
├── Scan-Processors-ASG:
│   ├── Min: 1 instance
│   ├── Max: 10 instances
│   ├── Target: Queue depth > 5 messages
│   └── Scale-out: +1 instance, 1-minute cooldown
└── Background-Workers-ASG:
    ├── Min: 1 instance
    ├── Max: 5 instances
    ├── Target: SQS queue age > 30 seconds
    └── Scale-out: +1 instance, 5-minute cooldown
```

## Horizontal Scaling of FastAPI Services
- **Load balancer**: Application Load Balancer with health checks
- **Service discovery**: AWS Cloud Map for dynamic service registration
- **Session management**: Stateless design with JWT tokens
- **Database connections**: Connection pooling with pgbouncer equivalent

## Caching Strategy (Redis Optional)
```python
class CacheStrategy:
    # L1: Application-level caching
    @lru_cache(maxsize=1000)
    def get_product_metadata(self, product_id: str):
        pass
    
    # L2: Redis distributed cache
    async def get_user_recommendations(self, user_id: str):
        cache_key = f"recommendations:{user_id}"
        cached = await redis.get(cache_key)
        if cached:
            return json.loads(cached)
        
        recommendations = await self._generate_recommendations(user_id)
        await redis.setex(cache_key, 300, json.dumps(recommendations))
        return recommendations
```

## Connection Pooling
```python
# DynamoDB connection pool
dynamodb_client = boto3.client(
    'dynamodb',
    config=Config(
        max_pool_connections=50,
        retries={'max_attempts': 3}
    )
)

# OpenSearch connection pool
opensearch_client = OpenSearch(
    hosts=[{'host': 'search-domain.region.es.amazonaws.com', 'port': 443}],
    http_auth=('username', 'password'),
    use_ssl=True,
    verify_certs=True,
    connection_class=RequestsHttpConnection,
    pool_maxsize=20
)
```

## Cold-Start Mitigation
- **Provisioned concurrency**: Lambda functions with pre-warmed containers
- **Connection pre-warming**: Database connections established at startup
- **Model pre-loading**: ML models loaded into memory during initialization
- **Health check optimization**: Lightweight health endpoints

## Target Latency < 3 Seconds
```
Performance Budget:
├── Image upload: 500ms
├── Attribute extraction: 1000ms
├── Vector search: 300ms
├── RAG inference: 800ms
├── Response formatting: 200ms
└── Network overhead: 200ms
Total: 3000ms
```

# 8. Privacy & Compliance Design

## In-Memory Image Lifecycle
```python
class ImageProcessor:
    def __init__(self):
        self.max_memory_mb = 100
        self.processing_timeout = 30
    
    async def process_image(self, image_data: bytes) -> Attributes:
        # Memory allocation tracking
        memory_tracker = MemoryTracker()
        
        try:
            # Load image to memory buffer
            image_buffer = memory_tracker.allocate(image_data)
            
            # Set processing timeout
            async with asyncio.timeout(self.processing_timeout):
                attributes = await self._extract_attributes(image_buffer)
            
            return attributes
        
        finally:
            # Guaranteed cleanup
            memory_tracker.deallocate_all()
            gc.collect()  # Force garbage collection
```

## Attribute-Only Persistence
```python
class AttributePersistence:
    ALLOWED_ATTRIBUTES = {
        'foot': ['length_mm', 'width_mm', 'arch_type', 'size_range'],
        'face': ['skin_tone', 'skin_type', 'age_range', 'climate_suitability'],
        'body': ['measurements', 'body_type', 'fit_preferences']
    }
    
    def sanitize_attributes(self, raw_attributes: Dict) -> Dict:
        sanitized = {}
        for attr_type, attributes in raw_attributes.items():
            if attr_type in self.ALLOWED_ATTRIBUTES:
                sanitized[attr_type] = {
                    k: v for k, v in attributes.items()
                    if k in self.ALLOWED_ATTRIBUTES[attr_type]
                }
        return sanitized
```

## Consent Logging Workflow
```python
class ConsentManager:
    async def log_consent(
        self,
        user_id: str,
        consent_type: str,
        granted: bool,
        request_context: RequestContext
    ):
        consent_record = {
            'user_id': user_id,
            'consent_timestamp': datetime.utcnow().isoformat(),
            'consent_type': consent_type,
            'consent_granted': granted,
            'consent_version': self.current_consent_version,
            'ip_address': self._hash_ip(request_context.client_ip),
            'user_agent': request_context.user_agent,
            'expiry_date': self._calculate_expiry(consent_type)
        }
        
        await self.dynamodb.put_item(
            TableName='consent-logs',
            Item=consent_record
        )
```

## GDPR & Indian DPDP Compliance Mapping
```
Compliance Matrix:
├── Data Minimization:
│   ├── GDPR Article 5(1)(c): Only necessary attributes stored
│   └── DPDP Section 5: Purpose limitation enforced
├── Consent Management:
│   ├── GDPR Article 7: Explicit consent with withdrawal
│   └── DPDP Section 11: Clear consent mechanisms
├── Right to Erasure:
│   ├── GDPR Article 17: Complete data deletion
│   └── DPDP Section 18: Right to erasure implementation
└── Data Protection by Design:
    ├── GDPR Article 25: Privacy by design principles
    └── DPDP Section 22: Data protection safeguards
```

## Audit Logging Design
```python
class AuditLogger:
    async def log_data_access(
        self,
        user_id: str,
        data_type: str,
        operation: str,
        request_id: str
    ):
        audit_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'user_id': self._hash_user_id(user_id),
            'data_type': data_type,
            'operation': operation,
            'request_id': request_id,
            'service': 'ecommerce-decision-engine',
            'compliance_flags': self._get_compliance_flags(operation)
        }
        
        # Immutable audit log
        await self.cloudwatch.put_log_events(
            logGroupName='/aws/compliance/audit-trail',
            logStreamName=f'data-access-{datetime.utcnow().strftime("%Y-%m-%d")}',
            logEvents=[{
                'timestamp': int(time.time() * 1000),
                'message': json.dumps(audit_entry)
            }]
        )
```

# 9. Continuous Learning Architecture

## Feedback Ingestion
```python
class FeedbackIngestionPipeline:
    def __init__(self):
        self.sqs_queue = 'feedback-processing-queue'
        self.batch_size = 100
        self.processing_interval = 300  # 5 minutes
    
    async def ingest_feedback(self, feedback: UserFeedback):
        # Immediate storage
        await self.store_feedback(feedback)
        
        # Queue for batch processing
        await self.sqs.send_message(
            QueueUrl=self.sqs_queue,
            MessageBody=json.dumps(feedback.dict()),
            MessageAttributes={
                'feedback_type': {'StringValue': feedback.type},
                'priority': {'StringValue': feedback.priority}
            }
        )
```

## Data Preprocessing
```python
class FeedbackPreprocessor:
    def preprocess_batch(self, feedback_batch: List[UserFeedback]) -> TrainingData:
        processed_data = []
        
        for feedback in feedback_batch:
            # Anonymize user data
            anonymized_user = self.anonymize_user(feedback.user_id)
            
            # Extract features
            features = self.extract_features(feedback)
            
            # Create training example
            training_example = {
                'user_cluster': anonymized_user.cluster_id,
                'product_features': features.product,
                'interaction_features': features.interaction,
                'outcome': feedback.outcome,
                'satisfaction_score': feedback.satisfaction_rating
            }
            
            processed_data.append(training_example)
        
        return TrainingData(examples=processed_data)
```

## Model Retraining Trigger
```python
class ModelRetrainingOrchestrator:
    def __init__(self):
        self.min_feedback_threshold = 1000
        self.accuracy_degradation_threshold = 0.05
        self.retraining_schedule = "0 2 * * 0"  # Weekly at 2 AM Sunday
    
    async def check_retraining_conditions(self):
        # Condition 1: Sufficient new feedback
        new_feedback_count = await self.get_new_feedback_count()
        
        # Condition 2: Model performance degradation
        current_accuracy = await self.evaluate_current_model()
        baseline_accuracy = await self.get_baseline_accuracy()
        
        # Condition 3: Scheduled retraining
        is_scheduled = self.is_scheduled_retraining_time()
        
        should_retrain = (
            new_feedback_count >= self.min_feedback_threshold or
            (baseline_accuracy - current_accuracy) > self.accuracy_degradation_threshold or
            is_scheduled
        )
        
        if should_retrain:
            await self.trigger_retraining()
```

## Model Versioning Strategy
```python
class ModelVersionManager:
    def __init__(self):
        self.model_registry = "s3://model-registry-bucket/"
        self.version_format = "v{major}.{minor}.{patch}"
    
    async def deploy_new_model(self, model_artifacts: ModelArtifacts):
        # Version the new model
        new_version = self.increment_version()
        
        # Upload to registry
        model_path = f"{self.model_registry}{new_version}/"
        await self.upload_model(model_artifacts, model_path)
        
        # Canary deployment (5% traffic)
        await self.deploy_canary(new_version, traffic_percentage=5)
        
        # Monitor performance
        await self.monitor_canary_performance(new_version)
```

## Rollback Mechanism
```python
class ModelRollbackManager:
    async def rollback_to_previous_version(self, reason: str):
        # Get current and previous versions
        current_version = await self.get_current_version()
        previous_version = await self.get_previous_stable_version()
        
        # Immediate traffic switch
        await self.switch_traffic(
            from_version=current_version,
            to_version=previous_version,
            percentage=100
        )
        
        # Log rollback event
        await self.log_rollback_event(
            from_version=current_version,
            to_version=previous_version,
            reason=reason,
            timestamp=datetime.utcnow()
        )
        
        # Alert operations team
        await self.send_rollback_alert(reason)
```

## A/B Testing Support
```python
class ABTestingFramework:
    def __init__(self):
        self.experiment_config = ExperimentConfig()
    
    async def assign_user_to_variant(self, user_id: str, experiment_id: str) -> str:
        # Consistent hash-based assignment
        hash_input = f"{user_id}:{experiment_id}"
        hash_value = hashlib.md5(hash_input.encode()).hexdigest()
        hash_int = int(hash_value[:8], 16)
        
        # Get experiment configuration
        experiment = await self.get_experiment(experiment_id)
        
        # Assign to variant based on traffic allocation
        cumulative_percentage = 0
        for variant in experiment.variants:
            cumulative_percentage += variant.traffic_percentage
            if (hash_int % 100) < cumulative_percentage:
                return variant.name
        
        return experiment.control_variant
```

# 10. Deployment Architecture

## CI/CD Pipeline (GitHub → AWS)
```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Tests
        run: |
          python -m pytest tests/
          npm run test:frontend
  
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Security Scan
        run: |
          bandit -r backend/
          npm audit --audit-level high
  
  deploy-staging:
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: |
          terraform apply -var="environment=staging"
  
  deploy-production:
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Production
        run: |
          terraform apply -var="environment=production"
```

## Infrastructure as Code (Terraform)
```hcl
# main.tf
module "vpc" {
  source = "./modules/vpc"
  
  cidr_block = var.vpc_cidr
  environment = var.environment
}

module "ecs_cluster" {
  source = "./modules/ecs"
  
  cluster_name = "${var.project_name}-${var.environment}"
  vpc_id = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}

module "api_gateway" {
  source = "./modules/api_gateway"
  
  stage_name = var.environment
  lambda_functions = module.lambda.function_arns
}

module "dynamodb" {
  source = "./modules/dynamodb"
  
  tables = [
    {
      name = "user-attributes"
      hash_key = "user_id"
      range_key = "attribute_type"
    },
    {
      name = "product-metadata"
      hash_key = "product_id"
    }
  ]
}
```

## Blue-Green Deployment Strategy
```python
class BlueGreenDeployment:
    def __init__(self):
        self.route53_client = boto3.client('route53')
        self.elbv2_client = boto3.client('elbv2')
    
    async def deploy_new_version(self, new_version_target_group: str):
        # Phase 1: Deploy to green environment
        await self.deploy_to_green_environment(new_version_target_group)
        
        # Phase 2: Health checks
        health_check_passed = await self.run_health_checks(new_version_target_group)
        
        if not health_check_passed:
            await self.rollback_green_deployment()
            raise DeploymentError("Health checks failed")
        
        # Phase 3: Gradual traffic shift
        await self.shift_traffic_gradually(
            from_target_group="blue-tg",
            to_target_group=new_version_target_group,
            steps=[10, 25, 50, 75, 100]
        )
        
        # Phase 4: Monitor and validate
        await self.monitor_deployment_metrics()
```

## Environment Separation (Dev / Staging / Prod)
```
Environment Configuration:
├── Development:
│   ├── Single AZ deployment
│   ├── t3.micro instances
│   ├── DynamoDB on-demand
│   └── Basic monitoring
├── Staging:
│   ├── Multi-AZ deployment
│   ├── t3.small instances
│   ├── Production-like data volume
│   └── Full monitoring suite
└── Production:
    ├── Multi-region deployment
    ├── c5.large instances
    ├── DynamoDB provisioned capacity
    └── Enhanced monitoring + alerting
```

# 11. Failure Handling & Resilience

## Circuit Breaker Design
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    async def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError("Circuit breaker is OPEN")
        
        try:
            result = await func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            
            raise e
```

## Retry Logic
```python
class RetryPolicy:
    def __init__(self):
        self.max_retries = 3
        self.base_delay = 1.0
        self.max_delay = 60.0
        self.exponential_base = 2
        self.jitter = True
    
    async def execute_with_retry(self, func, *args, **kwargs):
        for attempt in range(self.max_retries + 1):
            try:
                return await func(*args, **kwargs)
            
            except RetryableError as e:
                if attempt == self.max_retries:
                    raise e
                
                delay = min(
                    self.base_delay * (self.exponential_base ** attempt),
                    self.max_delay
                )
                
                if self.jitter:
                    delay *= (0.5 + random.random() * 0.5)
                
                await asyncio.sleep(delay)
            
            except NonRetryableError:
                raise
```

## Graceful Degradation (Fallback Recommendations)
```python
class RecommendationService:
    """
    Maintains high-confidence recommendation approach even during service degradation.
    Always prioritizes quality over quantity in recommendations.
    """
    
    async def get_recommendations(self, user_id: str) -> List[Product]:
        try:
            # Primary: AI-powered recommendations with visual input analysis
            # Uses GPT-4o + RAG with brand-specific knowledge grounding
            return await self.get_ai_recommendations(user_id)
        
        except AIServiceUnavailableError:
            # Fallback 1: Rule-based high-confidence matching
            # Uses stored visual attributes + structured product data
            try:
                user_attributes = await self.get_user_attributes(user_id)
                return await self.get_rule_based_high_confidence_matches(user_attributes)
            
            except AttributeServiceError:
                # Fallback 2: Collaborative filtering with confidence threshold
                # Still maintains >75% confidence requirement
                try:
                    return await self.get_collaborative_high_confidence_recommendations(user_id)
                
                except CollaborativeServiceError:
                    # Fallback 3: Popular products in user's category with availability filter
                    # Reduced confidence but still filtered for quality
                    try:
                        user_preferences = await self.get_user_preferences(user_id)
                        return await self.get_popular_high_quality_products(
                            user_preferences.categories,
                            min_rating=4.0  # Quality threshold
                        )
                    
                    except Exception:
                        # Final fallback: Generic trending products with quality filter
                        return await self.get_trending_quality_products(min_rating=4.5)
```

## Partial System Outage Handling
```python
class SystemHealthManager:
    def __init__(self):
        self.service_health = {
            'bedrock': True,
            'opensearch': True,
            'dynamodb': True,
            's3': True
        }
    
    async def handle_service_outage(self, service_name: str):
        self.service_health[service_name] = False
        
        if service_name == 'bedrock':
            # Disable AI recommendations, use rule-based fallback
            await self.enable_rule_based_recommendations()
        
        elif service_name == 'opensearch':
            # Use DynamoDB for basic product search
            await self.enable_basic_search_mode()
        
        elif service_name == 'dynamodb':
            # Use cached data and read-only mode
            await self.enable_read_only_mode()
        
        # Alert operations team
        await self.send_outage_alert(service_name)
```

# 12. Cost Optimization Strategy

## EC2 Instance Sizing
```python
class InstanceOptimizer:
    def __init__(self):
        self.instance_recommendations = {
            'api_servers': {
                'dev': 't3.micro',
                'staging': 't3.small',
                'prod': 'c5.large'
            },
            'scan_processors': {
                'dev': 't3.small',
                'staging': 't3.medium',
                'prod': 'c5.xlarge'
            }
        }
    
    def get_optimal_instance_type(self, service: str, environment: str, cpu_utilization: float):
        base_instance = self.instance_recommendations[service][environment]
        
        if cpu_utilization > 80:
            return self.scale_up_instance(base_instance)
        elif cpu_utilization < 30:
            return self.scale_down_instance(base_instance)
        
        return base_instance
```

## Bedrock Usage Optimization
```python
class BedrockCostOptimizer:
    def __init__(self):
        self.token_limits = {
            'input_tokens': 4000,
            'output_tokens': 1000
        }
        self.cache_duration = 300  # 5 minutes
    
    async def optimize_prompt(self, user_query: str, context: str) -> str:
        # Compress context to reduce token usage
        compressed_context = await self.compress_context(context)
        
        # Use template to minimize tokens
        optimized_prompt = self.build_efficient_prompt(
            user_query, 
            compressed_context
        )
        
        # Cache frequent queries
        cache_key = hashlib.md5(optimized_prompt.encode()).hexdigest()
        cached_response = await self.get_cached_response(cache_key)
        
        if cached_response:
            return cached_response
        
        return optimized_prompt
```

## OpenSearch Cost Tuning
```yaml
# OpenSearch cluster configuration
OpenSearchCluster:
  InstanceType: t3.small.search  # Cost-effective for development
  InstanceCount: 3
  DedicatedMasterEnabled: false  # Save costs for small clusters
  EBSEnabled: true
  EBSVolumeType: gp3  # More cost-effective than gp2
  EBSVolumeSize: 20
  
  # Cost optimization settings
  AdvancedOptions:
    "indices.fielddata.cache.size": "40%"
    "indices.query.bool.max_clause_count": "1024"
```

## DynamoDB On-Demand vs Provisioned
```python
class DynamoDBCostOptimizer:
    def analyze_table_usage(self, table_name: str) -> str:
        # Get usage metrics for last 30 days
        metrics = self.cloudwatch.get_metric_statistics(
            Namespace='AWS/DynamoDB',
            MetricName='ConsumedReadCapacityUnits',
            Dimensions=[{'Name': 'TableName', 'Value': table_name}],
            StartTime=datetime.utcnow() - timedelta(days=30),
            EndTime=datetime.utcnow(),
            Period=3600,
            Statistics=['Average', 'Maximum']
        )
        
        avg_rcu = sum(point['Average'] for point in metrics['Datapoints']) / len(metrics['Datapoints'])
        max_rcu = max(point['Maximum'] for point in metrics['Datapoints'])
        
        # Calculate costs
        on_demand_cost = self.calculate_on_demand_cost(avg_rcu)
        provisioned_cost = self.calculate_provisioned_cost(max_rcu * 1.2)  # 20% buffer
        
        return "on-demand" if on_demand_cost < provisioned_cost else "provisioned"
```

## CloudFront Caching
```yaml
# CloudFront distribution configuration
CloudFrontDistribution:
  DefaultCacheBehavior:
    TargetOriginId: S3Origin
    ViewerProtocolPolicy: redirect-to-https
    CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad  # Managed-CachingOptimized
    TTL:
      DefaultTTL: 86400  # 24 hours
      MaxTTL: 31536000   # 1 year
    
  CacheBehaviors:
    - PathPattern: "/api/*"
      CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
      TTL:
        DefaultTTL: 0      # No caching for API calls
        MaxTTL: 0
    
    - PathPattern: "/static/*"
      CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6  # Managed-CachingOptimizedForUncompressedObjects
      TTL:
        DefaultTTL: 31536000  # 1 year for static assets
        MaxTTL: 31536000
```

# 13. Future Extensibility

## 3D Body Mesh Integration
```python
class BodyMeshProcessor:
    def __init__(self):
        self.depth_sensor_support = True
        self.mesh_resolution = "medium"  # low, medium, high
        self.processing_timeout = 45  # seconds
    
    async def process_3d_scan(self, depth_images: List[bytes]) -> BodyMesh:
        # Multi-view stereo reconstruction
        point_cloud = await self.reconstruct_point_cloud(depth_images)
        
        # Mesh generation
        mesh = await self.generate_mesh(point_cloud)
        
        # Measurement extraction
        measurements = await self.extract_precise_measurements(mesh)
        
        return BodyMesh(
            vertices=mesh.vertices,
            faces=mesh.faces,
            measurements=measurements,
            confidence=mesh.quality_score
        )
```

## Multi-Language AI
```python
class MultilingualAIService:
    def __init__(self):
        self.supported_languages = [
            'en', 'hi', 'bn', 'te', 'mr', 'ta', 'gu', 'kn', 'ml', 'pa'
        ]
        self.translation_service = AmazonTranslate()
    
    async def generate_recommendation(
        self, 
        user_attributes: UserAttributes,
        language: str = 'en'
    ) -> Recommendation:
        # Generate in English first
        english_recommendation = await self.bedrock_client.generate_recommendation(
            user_attributes, language='en'
        )
        
        if language != 'en':
            # Translate to target language
            translated_recommendation = await self.translation_service.translate(
                text=english_recommendation.explanation,
                source_language='en',
                target_language=language
            )
            
            english_recommendation.explanation = translated_recommendation
        
        return english_recommendation
```

## AR Try-On Support
```python
class ARTryOnService:
    def __init__(self):
        self.ar_engine = "WebXR"  # or ARCore/ARKit
        self.model_formats = ["glTF", "USDZ"]
    
    async def generate_ar_model(self, product_id: str, user_measurements: Dict) -> ARModel:
        # Get product 3D model
        base_model = await self.get_product_3d_model(product_id)
        
        # Adapt to user measurements
        fitted_model = await self.fit_model_to_user(base_model, user_measurements)
        
        # Optimize for AR rendering
        optimized_model = await self.optimize_for_ar(fitted_model)
        
        return ARModel(
            model_url=optimized_model.url,
            format=optimized_model.format,
            size_mb=optimized_model.size,
            fit_confidence=fitted_model.confidence
        )
```

## Seller Analytics Microservice
```python
class SellerAnalyticsService:
    def __init__(self):
        self.analytics_db = "seller_analytics"
        self.update_frequency = 3600  # 1 hour
    
    async def generate_seller_insights(self, seller_id: str) -> SellerInsights:
        # Product performance metrics
        product_metrics = await self.get_product_performance(seller_id)
        
        # Size accuracy analysis
        size_accuracy = await self.analyze_size_accuracy(seller_id)
        
        # Return rate analysis
        return_analysis = await self.analyze_returns(seller_id)
        
        # Recommendation optimization suggestions
        optimization_suggestions = await self.generate_optimization_suggestions(
            product_metrics, size_accuracy, return_analysis
        )
        
        return SellerInsights(
            product_performance=product_metrics,
            size_accuracy=size_accuracy,
            return_analysis=return_analysis,
            optimization_suggestions=optimization_suggestions,
            generated_at=datetime.utcnow()
        )
```