model Locations {
  id                String   @id @default(uuid()) // 1
  address           String?  // 3
  cityStateProvince String?  // 3
  country           String?  // 3
  continent         String?  // 2
  isAvailable       Boolean  // 1
  organizeId        String   // 1
}

model ConferenceDates {
  id          String   @id @default(uuid()) // 1
  organizedId String   // 1
  fromDate    DateTime? // 1
  toDate      DateTime? // 1
  type        String   // 2
  name        String   // 3
  isAvailable Boolean  // 1
}

model ConferenceOrganizations {
  id                      String   @id @default(uuid()) // 1
  year                    Int?     // 1
  accessType              String   // 2
  isAvailable             Boolean  // 1
  conferenceId            String   // 1

  publisher               String   // 3

  summerize               String   // 3
  callForPaper            String   // 3

  link                    String   // 1
  cfpLink                 String   // 1
  impLink                 String   // 1

  isLastest               Boolean? // 1

  information             String?  // 3
  cfpSummary              String?  // 3

  conferenceStartDate     String?  // 1
  conferenceEndDate       String?  // 1

  reviewType              String?  // 3
  reviewDetails           String?  // 3

  rebuttalAllowed         Boolean? // 1
  rebuttalDetails         String?  // 3

  submissionPolicy        String?  // 3
  publicationPolicy       String?  // 3
  withdrawalPolicy        String?  // 3

  registrationFee         String?  // 2

  sponsors                String?  // 3
}

<!-- Chỉ là bảng liên kết -->
<!-- model ConferenceTopics {
  id         String                  @id @default(uuid())
  organizeId String
  topicId    String
  createdAt  DateTime                @default(now()) @db.Timestamptz(6)
  updatedAt  DateTime                @updatedAt @db.Timestamptz(6)
  belongsTo  ConferenceOrganizations @relation(fields: [organizeId], references: [id])
  inTopic    Topics                  @relation(fields: [topicId], references: [id])
} -->

model Topics {
  id                 String   @id @default(uuid()) // 1
  name               String   // 3
}

model Conferences {
  id        String   @id @default(uuid()) // 1

  title     String   // 3
  acronym   String   // 2

  status    String   // 2
}

<!-- Chỉ là bảng liên kết -->
<!-- model ConferenceRanks {
  id                String @id @default(uuid()) // 1

  year              Int    // 1

  conferenceId      String // 1
  fieldOfResearchId String // 1
  rankId            String // 1
} -->

model FieldOfResearchs {
  id   String @id @default(uuid()) // 1

  name String // 3
  code String // 2
}

model Ranks {
  id       String @id @default(uuid()) // 1

  name     String // 2
  value    Int    // 1
  sourceId String // 1
}

model Sources {
  id   String  @id @default(uuid()) // 1
  name String  @unique              // 2
  link String?                      // 1
}

model ConferenceBlacklists {
  id           String   @id @default  (uuid()) // 1

  conferenceId String   // 1
  userId       String   // 1

  createdAt    DateTime @default(now()) @db.Timestamptz(6) // 1
  updatedAt    DateTime @updatedAt @db.Timestamptz(6)      // 1
}