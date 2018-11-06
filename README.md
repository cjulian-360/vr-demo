# VR Portal Sprint Review

- [Work Completed](#work-completed)
- [Work Remaining](#work-remaining)
- [Query Examples](#query-examples)
  - [getDealerInfo](#getdealerinfo)
  - [getDealerDistributors](#getdealerdistributors)
  - [getDealersAccumulations](#getdealersAccumulations)
  - [Query Notes](#query-notes)
- [Web-API to Egg-GRPC](#web-api-to-egg-grpc)

## Work Completed
- converted all vr-egg queries to use ADM (almost)
- converted vr-egg to a gRPC service
- added vr-web Rest API that calls vr-egg gRPC service

## Work Remaining
- dealer list query (in progress, we ran into difficulty getting last invoice date)
- modify vr-web to call vr-web API instead of vr-egg API
- identify data that needs to be added to ADM to exercise all use cases supported by vr-web
- query optimization

## Query Examples

### getDealerInfo
```SQL
-- old
SELECT RW.External_id AS externalId
  , RW.Internal_id AS internalId
  , RW.LegalName AS name
  , MAX(RUS.maxOfSaleDate) AS lastInvoiceDate
  , RWCL.Address1
  , RWCL.Address2
  , RWCL.City
  , RWCL.State
  , RWCL.ZipCode
  , RWCL.Country
  , RWCL.PrimaryContactEmail
  , RWCL.PrimaryContactFirstName
  , RWCL.PrimaryContactLastName
  , RWCL.PrimaryContactHomePhone
  , RWCL.PrimaryContactBusinessPhone
FROM Rewardee RW
    LEFT JOIN Rewardee_CoreLookup RWCL ON RW.External_id = RWCL.External_id
    LEFT JOIN RewardeeUnitSummary RUS ON RUS.dealerExternalId = RW.External_id
WHERE
    RW.External_id = :dealerId
```

```SQL
- new
SELECT
  rs.resellerid as internalid
  , org.legalname
  , sa.addressline1 as address1
  , sa.addressline2 as address2
  , sa.city as city
  , sa.subdivisionname as state
  , sa.postalcode as zipcode
  , sa.country as country
  , org.primarytelephonenumber
  , per.primaryemailaddress
  , per.firstname
  , per.lastname
FROM dimreseller rs
INNER JOIN dimxref xrfrstype ON
  xrfrstype.xrefidkey = rs.resellertypexrefkey
  AND xrfrstype.isactive = true
  AND xrfrstype.expireddatetime IS NULL
  AND xrfrstype.value = 'Dealer'
INNER JOIN dimorganization org ON
  org.organizationkey = rs.organizationkey
  AND org.isactive = true
  AND org.expireddatetime IS NULL
LEFT JOIN dimaddress sa ON
  sa.addresskey = org.primarystreetaddresskey
  AND sa.isactive = true
  AND sa.expireddatetime IS NULL
INNER JOIN dimsalesorglevel sol ON
  sol.salesorgkey = rs.salesorgkey
  AND sol.isactive = true
  AND sol.expireddatetime IS NULL
LEFT JOIN dimparty prs ON
  prs.partykey = org.partykey
  AND prs.isactive = true
  AND prs.expireddatetime IS NULL
LEFT JOIN dimpartyaffiliation pa ON
  pa.partykey = prs.partykey
  AND pa.isactive = true
  AND pa.expireddatetime IS NULL
  AND pa.partyaffiliationtypexrefkey IN (
    SELECT
        dimxref.xrefidkey
    FROM dimxref
    WHERE
        dimxref.value = 'PrimaryContact'
        AND dimxref.xreferenceset = 'PartyAffiliationType'
        AND dimxref.isactive = true
        AND dimxref.expireddatetime IS NULL
  )
LEFT JOIN dimparty pp ON
  pp.partykey = pa.subpartykey
  AND pp.isactive = true
  AND pp.expireddatetime IS NULL
LEFT JOIN dimperson per ON
  per.partykey = pp.partykey
  AND per.isactive = true
  AND per.expireddatetime IS NULL
INNER JOIN dimsaleschannellevel scl ON
  scl.salesorglevelkey = sol.salesorglevelkey
  AND scl.isactive = true
  AND scl.expireddatetime IS NULL
INNER JOIN dimsaleschannel sc ON
  sc.saleschannelkey = scl.saleschannelkey
  AND sc.isactive = true
  AND sc.expireddatetime IS NULL
INNER JOIN dimrebatesponsor rsp ON
  rsp.rebatesponsorkey = sc.rebatesponsorkey
  AND rsp.isactive = true
  AND rsp.expireddatetime IS NULL
  AND rsp.rebatesponsorid = :rebateSponsorId
WHERE
  rs.isactive = true
  AND rs.expireddatetime IS NULL
  AND rs.resellerid = :dealerResellerId
```

### getDealerDistributors
```SQL
-- old
SELECT RS.External_id
    , RS.LegalName
    , RSCL.Address1
    , RSCL.Address2
    , RSCL.City
    , RSCL.State
    , RSCL.ZipCode
    , RSCL.Country
    , RRR.RelationshipType
    , RRR.RelationshipSet
FROM ResellerRewardee_Relationship RRR
LEFT JOIN Reseller RS ON
    RS.External_id = RRR.ResellerExternal_id
LEFT JOIN Reseller_CoreLookup RSCL ON
    RS.External_id = RSCL.External_id
INNER JOIN User_CoreLookup UCL ON
    UCL.Internal_id = :legacyId
    AND (
        UCL.ResellerInternal_id IS NULL
        OR RSCL.Internal_id = UCL.ResellerInternal_id
    )
WHERE
    RRR.RewardeeExternal_id = :dealerId
    AND RRR.Active = 1
    AND NOW() >= RRR.EffectiveDate
    AND (
        RRR.InEffectiveDate IS NULL
        OR NOW() <= RRR.InEffectiveDate
    )
    AND RRR.IsException = 0
```

```SQL
-- new
SELECT
    xrscatdist.value as relationshiptype
    , rsdist.resellerid as internalid
    , org.legalname as legalname
    , sa.addressline1 as address1
    , sa.addressline2 as address2
    , sa.city as city
    , sa.subdivisionname as state
    , sa.postalcode as zipcode
    , sa.country as country
FROM dimreseller rsdealer
INNER JOIN dimsaleschannellevel scldealer ON
    scldealer.channelparticipantkey = rsdealer.channelparticipantkey
    AND scldealer.isactive = true
    AND scldealer.expireddatetime IS NULL
INNER JOIN dimxref xrscprdealer ON
    xrscprdealer.xrefidkey = scldealer.saleschannelparticipantrolexrefkey
    AND xrscprdealer.isactive = true
    AND xrscprdealer.expireddatetime IS NULL
    AND xrscprdealer.value = 'Dealer'
INNER JOIN BridgeSalesChannelLevelHierarchy sclhdealerdist ON
    sclhdealerdist.saleschannellevelkey = scldealer.saleschannellevelkey
    AND sclhdealerdist.isactive = true
INNER JOIN dimsaleschannellevel scldist ON
    scldist.saleschannellevelkey = sclhdealerdist.parentsaleschannellevelkey
    AND scldealer.isactive = true
    AND scldealer.expireddatetime IS NULL
INNER JOIN dimsaleschannel sc ON
    sc.saleschannelkey = scldist.saleschannelkey
    AND sc.saleschannelkey = scldealer.saleschannelkey
    AND sc.isactive = true
    AND sc.expireddatetime IS NULL
INNER JOIN dimrebatesponsor rs ON
    rs.rebatesponsorkey = sc.rebatesponsorkey
    AND rs.isactive = true
    AND rs.expireddatetime IS NULL
    AND rs.rebatesponsorid = :rebateSponsorId
INNER JOIN dimxref xrscprdist ON
    xrscprdist.xrefidkey = scldist.saleschannelparticipantrolexrefkey
    AND xrscprdist.isactive = true
    AND xrscprdist.expireddatetime IS NULL
    AND xrscprdist.value = 'Distributor'
INNER JOIN dimxref xrscatdist ON
    xrscatdist.xrefidkey = scldist.saleschannelassignmenttypexrefkey
    AND xrscatdist.isactive = true
    AND xrscatdist.expireddatetime IS NULL
INNER JOIN dimreseller rsdist ON
    rsdist.channelparticipantkey = scldist.channelparticipantkey
    AND rsdist.isactive = true
    AND rsdist.expireddatetime IS NULL
INNER JOIN dimxref xrdisttype ON
    xrdisttype.xrefidkey = rsdist.resellertypexrefkey
    AND xrdisttype.value = 'Distributor'
    AND xrdisttype.isactive = true
    AND xrdisttype.expireddatetime IS NULL
INNER JOIN dimorganization org ON
    org.organizationkey = rsdist.organizationkey
    AND org.isactive = true
    AND org.expireddatetime IS NULL
LEFT JOIN dimaddress sa ON
    sa.addresskey = org.primarystreetaddresskey
    AND sa.isactive = true
    AND sa.expireddatetime IS NULL
WHERE
    rsdealer.isactive = true
    AND rsdealer.expireddatetime IS NULL
    AND rsdealer.resellerid = :dealerResellerId
```

### getDealersAccumulations
```SQL
-- old
SELECT DISTINCT
  relationship_types.Priority AS priority,
  IFNULL(units_per_relationship.total, 0) AS total,
  IFNULL(units_per_relationship.totalEligible, 0) AS qualified
FROM
  Relationship_AllowedRate relationship_types
LEFT JOIN
  RewardeePeriodAllowedUnits AS units_per_relationship
ON
  units_per_relationship.RelationshipTypeId = relationship_types.RelationshipType
  AND units_per_relationship.RewardeeExternal_id = :dealerId
  AND units_per_relationship.PeriodId = :periodId
```

```SQL
-- new
SELECT
  fvm.accumulativequantity,
  fvm_max.name AS accumulatorname
FROM dimvolumerewardeeacct dvra
INNER JOIN (
  SELECT
    MAX(vm.sequencenumber) AS sequencenumber,
    vm.volumerewardeeacctkey,
    vm.measuretypeannotationkey,
    ea.name
  FROM
    factvolumemeasure vm
  JOIN dimentityannotationmetadata ea
    ON ea.entityannotationmetadatakey = vm.measuretypeannotationkey
    AND ea.name IN (:accumulatorParams)
    AND ea.isactive = true
    AND vm.isactive = true
  GROUP BY
    vm.volumerewardeeacctkey,
    vm.measuretypeannotationkey,
    ea.name
) fvm_max ON fvm_max.volumerewardeeacctkey = dvra.volumerewardeeacctkey
INNER JOIN factvolumemeasure fvm
  ON fvm.volumerewardeeacctkey = fvm_max.volumerewardeeacctkey
  AND fvm.measuretypeannotationkey = fvm_max.measuretypeannotationkey
  AND fvm.sequencenumber = fvm_max.sequencenumber
  AND fvm.isactive = true
INNER JOIN dimpromotionenrollment dpe
  ON dpe.promotionenrollmentkey = fvm.promotionenrollmentkey
  AND dpe.isactive = true
INNER JOIN dimpromotionaloffer dpo
  ON dpo.promotionalofferkey = dpe.promotionalofferkey
  AND dpo.isactive = true
  AND dpo.expireddatetime IS NULL
INNER JOIN dimreseller dr
  ON dr.channelparticipantkey = dpe.channelparticipantkey
  AND dr.isactive = true
  AND dr.expireddatetime IS NULL
INNER JOIN dimxref dxr
  ON dxr.xrefidkey = dr.resellertypexrefkey
  AND dxr.xreferenceset = 'ResellerType'
  AND dxr.value = 'Dealer'
  AND dxr.isactive = true
INNER JOIN dimperiod dp
  ON dp.periodkey = dvra.periodkey
WHERE
  dvra.isactive = true
  AND dp.periodid = :periodId
  AND dpo.promotionalofferid = :promotionalOfferId
  AND dr.resellerid = :dealerResellerId
```

### Query Notes
- in general new queries were significantly longer with more joins
- old queries don't map one-to-one with new:
  - introduced a Data Access Layer with queries to retrieve commonly needed sets of data
  - introduced a Business Logic Layer that stitched together results from multipl queries
- shout-out to DNA team for helping us out

## Web-API to Egg-GRPC
Add a few screenshots of Postman to showcase end-to-end request from vr-web API to vr-egg GRPC + ADM
