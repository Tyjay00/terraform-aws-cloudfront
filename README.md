# Terraform AWS CloudFront Module 🌐

> **Global content delivery network with edge caching, SSL termination, and performance optimization**

[![Terraform](https://img.shields.io/badge/Terraform-%E2%89%A5%201.3-623CE4?logo=terraform)](https://terraform.io)
[![AWS Provider](https://img.shields.io/badge/AWS%20Provider-%E2%89%A5%205.0-FF9900?logo=amazon-aws)](https://registry.terraform.io/providers/hashicorp/aws/latest)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## 🎯 **Overview**

This Terraform module deploys AWS CloudFront distributions with custom domains, SSL certificates, caching policies, and security configurations. Ideal for accelerating websites, APIs, and content delivery with global edge locations.

## 🚀 **Features**

### **Core CDN Capabilities**
- 🌍 **Global Edge Network** - 400+ edge locations worldwide
- ⚡ **Performance Optimization** - Intelligent caching and compression
- 🔒 **SSL/TLS Termination** - Custom certificates and security policies
- 🎯 **Custom Domains** - Professional branding and CNAME support
- 📊 **Real-time Metrics** - Performance and usage analytics
- 🛡️ **DDoS Protection** - Built-in AWS Shield integration

### **Advanced Features**
- 🔐 **Origin Access Control** - Secure S3 bucket access
- 🚀 **HTTP/2 Support** - Modern protocol optimization
- 📈 **Lambda@Edge** - Serverless edge computing
- 🎯 **Geo Restrictions** - Country-based access control
- 🔄 **Cache Behaviors** - Granular caching rules
- 📝 **Custom Error Pages** - Branded error handling

## 📋 **Usage**

### **Basic S3 Website Distribution**
```hcl
module "website_cdn" {
  source = "./terraform-aws-cloudfront"

  # Origin Configuration
  origin_domain_name = "my-website-bucket.s3.amazonaws.com"
  origin_id          = "S3-my-website-bucket"
  
  # Domain Configuration
  domain_name     = "www.example.com"
  certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/12345"
  
  # Caching
  default_cache_behavior = {
    target_origin_id       = "S3-my-website-bucket"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    compress              = true
    cache_policy_id       = "658327ea-f89d-4fab-a63d-7e88639e58f6"  # Managed-CachingOptimized
  }
  
  project_name = "company-website"
  environment  = "production"
}
```

### **API Gateway with Custom Caching**
```hcl
module "api_cdn" {
  source = "./terraform-aws-cloudfront"

  # Multiple Origins
  origins = [
    {
      domain_name = "api.example.com"
      origin_id   = "API-Gateway"
      custom_origin_config = {
        http_port              = 80
        https_port             = 443
        origin_protocol_policy = "https-only"
        origin_ssl_protocols   = ["TLSv1.2"]
      }
    }
  ]
  
  # Domain Configuration
  domain_name     = "cdn.example.com"
  certificate_arn = module.acm_certificate.certificate_arn
  
  # Default Cache Behavior
  default_cache_behavior = {
    target_origin_id       = "API-Gateway"
    viewer_protocol_policy = "https-only"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD", "OPTIONS"]
    compress              = true
    
    forwarded_values = {
      query_string = true
      headers      = ["Authorization", "Content-Type"]
      cookies = {
        forward = "none"
      }
    }
    
    min_ttl     = 0
    default_ttl = 300     # 5 minutes
    max_ttl     = 31536000  # 1 year
  }
  
  # Ordered Cache Behaviors
  ordered_cache_behaviors = [
    {
      path_pattern           = "/api/v1/*"
      target_origin_id       = "API-Gateway"
      viewer_protocol_policy = "https-only"
      allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
      cached_methods         = ["GET", "HEAD", "OPTIONS"]
      compress              = true
      
      forwarded_values = {
        query_string = true
        headers      = ["*"]
        cookies = {
          forward = "all"
        }
      }
      
      min_ttl     = 0
      default_ttl = 0       # No caching for API
      max_ttl     = 0
    }
  ]
  
  project_name = "api-service"
  environment  = "production"
}
```

### **Multi-Origin Distribution with Lambda@Edge**
```hcl
module "enterprise_cdn" {
  source = "./terraform-aws-cloudfront"

  # Multiple Origins
  origins = [
    {
      domain_name = "static.example.com"
      origin_id   = "S3-Static"
      s3_origin_config = {
        origin_access_identity = aws_cloudfront_origin_access_identity.static.cloudfront_access_identity_path
      }
    },
    {
      domain_name = "api.example.com"
      origin_id   = "API-Gateway"
      custom_origin_config = {
        http_port              = 80
        https_port             = 443
        origin_protocol_policy = "https-only"
        origin_ssl_protocols   = ["TLSv1.2"]
      }
    },
    {
      domain_name = "media.example.com"
      origin_id   = "S3-Media"
      s3_origin_config = {
        origin_access_identity = aws_cloudfront_origin_access_identity.media.cloudfront_access_identity_path
      }
    }
  ]
  
  # Aliases
  aliases = [
    "cdn.example.com",
    "assets.example.com"
  ]
  
  # SSL Configuration
  viewer_certificate = {
    acm_certificate_arn      = module.acm_certificate.certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
  
  # Default Cache Behavior with Lambda@Edge
  default_cache_behavior = {
    target_origin_id       = "S3-Static"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    compress              = true
    cache_policy_id       = "658327ea-f89d-4fab-a63d-7e88639e58f6"
    
    lambda_function_associations = [
      {
        event_type   = "viewer-request"
        lambda_arn   = aws_lambda_function.edge_auth.qualified_arn
        include_body = false
      }
    ]
  }
  
  # Ordered Cache Behaviors
  ordered_cache_behaviors = [
    {
      path_pattern           = "/api/*"
      target_origin_id       = "API-Gateway"
      viewer_protocol_policy = "https-only"
      allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
      cached_methods         = ["GET", "HEAD", "OPTIONS"]
      cache_policy_id       = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"  # Managed-CachingDisabled
      origin_request_policy_id = "88a5eaf4-2fd4-4709-b370-b4c650ea3fcf"  # Managed-CORS-S3Origin
    },
    {
      path_pattern           = "/media/*"
      target_origin_id       = "S3-Media"
      viewer_protocol_policy = "https-only"
      allowed_methods        = ["GET", "HEAD"]
      cached_methods         = ["GET", "HEAD"]
      cache_policy_id       = "658327ea-f89d-4fab-a63d-7e88639e58f6"  # Managed-CachingOptimized
      
      lambda_function_associations = [
        {
          event_type   = "origin-response"
          lambda_arn   = aws_lambda_function.image_optimization.qualified_arn
          include_body = false
        }
      ]
    }
  ]
  
  # Custom Error Pages
  custom_error_responses = [
    {
      error_code         = 404
      response_code      = 200
      response_page_path = "/index.html"
      error_caching_min_ttl = 300
    },
    {
      error_code         = 403
      response_code      = 200
      response_page_path = "/index.html"
      error_caching_min_ttl = 300
    }
  ]
  
  # Geographic Restrictions
  geo_restriction = {
    restriction_type = "whitelist"
    locations        = ["US", "CA", "GB", "DE", "FR"]
  }
  
  # Security
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn
  
  project_name = "enterprise-platform"
  environment  = "production"
  
  tags = {
    Application = "enterprise-platform"
    Team        = "platform"
    CostCenter  = "engineering"
  }
}
```

## 📝 **Input Variables**

### **Required Variables**
| Name | Description | Type |
|------|-------------|------|
| `origin_domain_name` | Primary origin domain name | `string` |
| `origin_id` | Unique identifier for origin | `string` |

### **Distribution Configuration**
| Name | Description | Type | Default |
|------|-------------|------|---------|
| `aliases` | List of domain aliases | `list(string)` | `[]` |
| `enabled` | Enable the distribution | `bool` | `true` |
| `is_ipv6_enabled` | Enable IPv6 support | `bool` | `true` |
| `comment` | Distribution comment | `string` | `""` |
| `default_root_object` | Default root object | `string` | `"index.html"` |

### **SSL/TLS Configuration**
| Name | Description | Type | Default |
|------|-------------|------|---------|
| `certificate_arn` | ACM certificate ARN | `string` | `""` |
| `ssl_support_method` | SSL support method | `string` | `"sni-only"` |
| `minimum_protocol_version` | Minimum TLS version | `string` | `"TLSv1.2_2021"` |

### **Caching Configuration**
| Name | Description | Type | Default |
|------|-------------|------|---------|
| `price_class` | CloudFront price class | `string` | `"PriceClass_All"` |
| `default_cache_behavior` | Default cache behavior | `object` | See defaults |
| `ordered_cache_behaviors` | Ordered cache behaviors | `list(object)` | `[]` |

### **Security Features**
| Name | Description | Type | Default |
|------|-------------|------|---------|
| `web_acl_id` | WAF Web ACL ID | `string` | `""` |
| `geo_restriction` | Geographic restrictions | `object` | `null` |
| `viewer_protocol_policy` | Viewer protocol policy | `string` | `"redirect-to-https"` |

## 📤 **Outputs**

| Name | Description |
|------|-------------|
| `distribution_id` | CloudFront distribution ID |
| `distribution_arn` | CloudFront distribution ARN |
| `distribution_domain_name` | CloudFront domain name |
| `distribution_hosted_zone_id` | CloudFront hosted zone ID |
| `distribution_status` | Distribution status |
| `distribution_tags` | Distribution tags |
| `origin_access_identity_ids` | Origin access identity IDs |

## 🏗️ **Architecture**

```
                    ┌─────────────────┐
                    │    Global       │
                    │   Internet      │
                    └─────────┬───────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Edge Cache │     │  Edge Cache │     │  Edge Cache │
│  (US-East)  │     │  (Europe)   │     │  (Asia)     │
└─────────┬───┘     └─────────┬───┘     └─────────┬───┘
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
                    ┌─────────▼───────┐
                    │   CloudFront    │
                    │  Distribution   │
                    └─────────┬───────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│     S3      │     │     API     │     │   Lambda    │
│   Origin    │     │   Gateway   │     │   @Edge     │
└─────────────┘     └─────────────┘     └─────────────┘
```

## 🔒 **Security Best Practices**

### **Access Control**
- 🛡️ **Origin Access Control** - Secure S3 bucket access
- 🔒 **WAF Integration** - Web application firewall
- 🌍 **Geo Restrictions** - Country-based blocking
- 🔐 **Signed URLs** - Time-limited access tokens

### **SSL/TLS Security**
- 🔒 **TLS 1.2+ Only** - Modern encryption standards
- 📜 **Custom Certificates** - ACM integration
- 🛡️ **Security Headers** - Lambda@Edge security enhancement
- 📊 **HSTS Support** - HTTP Strict Transport Security

### **DDoS Protection**
- 🛡️ **AWS Shield Standard** - Automatic DDoS protection
- 🚨 **Rate Limiting** - Request throttling
- 📊 **Monitoring** - Real-time threat detection
- 🔄 **Auto-scaling** - Dynamic capacity management

## 💰 **Cost Optimization**

### **Pricing Components**
- **Data Transfer Out**: $0.085-$0.17 per GB (varies by region)
- **HTTP/HTTPS Requests**: $0.0075-$0.016 per 10,000 requests
- **Origin Requests**: $0.0075 per 10,000 requests
- **Field-Level Encryption**: $0.02 per 10,000 requests

### **Cost-Saving Strategies**
- 🎯 **Price Classes** - Regional distribution optimization
- 📊 **Cache Optimization** - Reduce origin requests
- 🔄 **Compression** - Minimize data transfer
- 📈 **Reserved Capacity** - Predictable high-volume discounts

## 🧪 **Examples**

Check the [examples](examples/) directory for complete implementations:

- **[Static Website](examples/static-website/)** - S3 website acceleration
- **[API Acceleration](examples/api-acceleration/)** - API Gateway caching
- **[Media Streaming](examples/media-streaming/)** - Video content delivery
- **[E-commerce Platform](examples/e-commerce/)** - Multi-origin distribution

## 🔧 **Requirements**

| Name | Version |
|------|---------|
| terraform | >= 1.3.0 |
| aws | >= 5.0 |

## 🧪 **Testing**

```bash
# Validate Terraform configuration
terraform validate

# Test distribution performance
curl -I https://your-distribution.cloudfront.net

# Check cache hit ratio
aws cloudfront get-distribution-config --id E1234567890123

# Performance testing
ab -n 1000 -c 10 https://your-distribution.cloudfront.net/
```

## 📊 **Performance Optimization**

### **Caching Strategies**
- 🎯 **TTL Optimization** - Balance freshness and performance
- 📊 **Cache Behaviors** - Different rules for different content
- 🔄 **Query String Handling** - Optimize cache key generation
- 📈 **Compression** - Gzip/Brotli content encoding

### **Edge Computing**
```hcl
# Lambda@Edge for dynamic content
resource "aws_lambda_function" "edge_optimization" {
  filename         = "edge-function.zip"
  function_name    = "cloudfront-edge-optimization"
  role            = aws_iam_role.lambda_edge.arn
  handler         = "index.handler"
  runtime         = "nodejs18.x"
  publish         = true
  
  # Lambda@Edge specific configuration
  timeout = 5
  memory_size = 128
}
```

## 🤝 **Contributing**

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/cdn-optimization`)
3. Commit your changes (`git commit -m 'Add CDN optimization'`)
4. Push to the branch (`git push origin feature/cdn-optimization`)
5. Open a Pull Request

## 📄 **License**

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🏆 **Related Modules**

- **[terraform-aws-s3](../terraform-aws-s3)** - Origin configuration
- **[terraform-aws-acm](../terraform-aws-acm)** - SSL certificates
- **[terraform-aws-route53](../terraform-aws-route53)** - DNS management
- **[terraform-aws-lambda](../terraform-aws-lambda)** - Edge functions

---

**🌐 Built for global content delivery and performance optimization**

> *This module demonstrates advanced CloudFront architecture patterns and CDN expertise suitable for high-traffic production environments requiring global scale and performance.*