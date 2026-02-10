# Prompt

Design a system for collecting Street View images from taxi-mounted cameras that 
can handle high-volume uploads, deduplication, and prepare images
for privacy/quality processing.

----

# What questions should I ask
1. scale -> requests per day / second
2. what is the format and size of the images ?
3. is the processing on the images multi step ?
4. what should be the retention policy for uploaded images ?
5. are the requests globally distributed, do we have any data residency requirements
    we need to think about ?
6. Is the deduplication at byte level or is it content aware ?
7. Should deduplication happen at storage time ?
8. how is the network between the taxi and the system ?
9. Is the upload and processing realtime or can it be batched ?
10. who is the consumer for processed images ?

# Functional Requirements
1. System should support uploading street view images over unreliable network
    across the globe. The images upload will asynchronous. 
2. System should support multi step processing of uploaded images.
3. System should support deduplicating while storing the processed images.


# Non Functional Requirements
1. System should be highly available (ingestion, processing, storage). SLA 99.99.
2. System should support durably storing the uploaded images. 
3. System should support fault tolerant processing of uploaded images. 

# Scale / Storage
Assume 100

# Entities / APIs
StreetViewImage
- id
- metadata: (location, orientation, timestamp, taxi_metadata, image_metadata)
- image: bytes

POST /v1/images
-> StreetViewImage(id, Metadata)
<- StreetViewImageUpload
    {
        image_id: 
        upload_url: presigned_url
        expires_in: 
    }







