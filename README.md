# User-Notification-Preferences-API-Challenge

 1. Project Setup


 1.1. Install Required Dependencies

1. Install Nest CLI (if not already installed):

bash
npm i -g @nestjs/cli


2. Create a new project:

bash
nest new notification-api


3. Install dependencies:

bash
cd notification-api
npm install @nestjs/mongoose mongoose class-validator class-transformer
npm install --save-dev jest @nestjs/testing


 2. User Preferences


 2.1. Create User Preference Schema


typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

@Schema()
export class UserPreference extends Document {
  @Prop({ required: true })
  userId: string;

  @Prop({ required: true })
  email: string;

  @Prop({ required: true })
  preferences: {
    marketing: boolean;
    newsletter: boolean;
    updates: boolean;
    frequency: 'daily' | 'weekly' | 'monthly' | 'never';
    channels: {
      email: boolean;
      sms: boolean;
      push: boolean;
    };
  };

  @Prop({ required: true })
  timezone: string;

  @Prop({ default: Date.now })
  createdAt: Date;

  @Prop({ default: Date.now })
  lastUpdated: Date;
}

export const UserPreferenceSchema = SchemaFactory.createForClass(UserPreference);


 2.2. Create DTO for User Preferences

Define a DTO to handle the creation and updates of user preferences:

typescript
// src/preferences/dto/create-preference.dto.ts
import { IsEmail, IsEnum, IsNotEmpty, IsObject, IsString } from 'class-validator';

export class CreateUserPreferenceDto {
  @IsNotEmpty()
  @IsString()
  userId: string;

  @IsNotEmpty()
  @IsEmail()
  email: string;

  @IsObject()
  preferences: {
    marketing: boolean;
    newsletter: boolean;
    updates: boolean;
    frequency: 'daily' | 'weekly' | 'monthly' | 'never';
    channels: {
      email: boolean;
      sms: boolean;
      push: boolean;
    };
  };

  @IsNotEmpty()
  @IsString()
  timezone: string;
}


 2.3. User Preferences Service

Create a service to handle CRUD operations on user preferences:

typescript
// src/preferences/preferences.service.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { UserPreference } from './schemas/user-preference.schema';
import { CreateUserPreferenceDto } from './dto/create-preference.dto';

@Injectable()
export class PreferencesService {
  constructor(
    @InjectModel(UserPreference.name) private userPreferenceModel: Model<UserPreference>,
  ) {}

  async create(createUserPreferenceDto: CreateUserPreferenceDto): Promise<UserPreference> {
    const createdPreference = new this.userPreferenceModel(createUserPreferenceDto);
    return createdPreference.save();
  }

  async findByUserId(userId: string): Promise<UserPreference> {
    return this.userPreferenceModel.findOne({ userId });
  }

  async update(userId: string, updateUserPreferenceDto: CreateUserPreferenceDto): Promise<UserPreference> {
    return this.userPreferenceModel.findOneAndUpdate({ userId }, updateUserPreferenceDto, { new: true });
  }

  async delete(userId: string): Promise<any> {
    return this.userPreferenceModel.findOneAndDelete({ userId });
  }
}


 2.4. User Preferences Controller

Define the controller for handling incoming requests:

typescript
// src/preferences/preferences.controller.ts
import { Controller, Get, Post, Patch, Delete, Body, Param } from '@nestjs/common';
import { PreferencesService } from './preferences.service';
import { CreateUserPreferenceDto } from './dto/create-preference.dto';
import { UserPreference } from './schemas/user-preference.schema';

@Controller('api/preferences')
export class PreferencesController {
  constructor(private readonly preferencesService: PreferencesService) {}

  @Post()
  async create(@Body() createUserPreferenceDto: CreateUserPreferenceDto): Promise<UserPreference> {
    return this.preferencesService.create(createUserPreferenceDto);
  }

  @Get(':userId')
  async findOne(@Param('userId') userId: string): Promise<UserPreference> {
    return this.preferencesService.findByUserId(userId);
  }

  @Patch(':userId')
  async update(@Param('userId') userId: string, @Body() updateUserPreferenceDto: CreateUserPreferenceDto): Promise<UserPreference> {
    return this.preferencesService.update(userId, updateUserPreferenceDto);
  }

  @Delete(':userId')
  async remove(@Param('userId') userId: string): Promise<any> {
    return this.preferencesService.delete(userId);
  }
}
```

### 3. **Notification Logs**

Next, we implement the **Notification Log** model to store the logs of sent notifications.

 3.1. Notification Log Schema

typescript
// src/notifications/schemas/notification-log.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

@Schema()
export class NotificationLog extends Document {
  @Prop({ required: true })
  userId: string;

  @Prop({ required: true })
  type: 'marketing' | 'newsletter' | 'updates';

  @Prop({ required: true })
  channel: 'email' | 'sms' | 'push';

  @Prop({ default: 'pending' })
  status: 'pending' | 'sent' | 'failed';

  @Prop({ required: true })
  metadata: Record<string, any>;

  @Prop()
  sentAt: Date;

  @Prop()
  failureReason?: string;
}

export const NotificationLogSchema = SchemaFactory.createForClass(NotificationLog);


 3.2. Notification Service

Create a service to handle the sending of notifications and logging:

```typescript
// src/notifications/notifications.service.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { NotificationLog } from './schemas/notification-log.schema';

@Injectable()
export class NotificationsService {
  constructor(
    @InjectModel(NotificationLog.name) private notificationLogModel: Model<NotificationLog>,
  ) {}

  async sendNotification(userId: string, type: string, channel: string, content: any): Promise<NotificationLog> {
    const notification = new this.notificationLogModel({
      userId,
      type,
      channel,
      metadata: content,
      status: 'pending',
    });
    await notification.save();

    setTimeout(async () => {
      notification.status = 'sent';
      notification.sentAt = new Date();
      await notification.save();
    }, 3000); 

    return notification;
  }

  async getNotificationLogs(userId: string) {
    return this.notificationLogModel.find({ userId });
  }

  async getStats() {
    const stats = await this.notificationLogModel.aggregate([
      { $group: { _id: '$status', count: { $sum: 1 } } },
    ]);
    return stats;
  }
}


 3.3. Notification Controller

Define the controller for handling notification sending and logs:

typescript
import { Controller, Post, Get, Body, Param } from '@nestjs/common';
import { NotificationsService } from './notifications.service';

@Controller('api/notifications')
export class NotificationsController {
  constructor(private readonly notificationsService: NotificationsService) {}

  @Post('send')
  async sendNotification(@Body() body: { userId: string; type: string; channel: string; content: any }) {
    return this.notificationsService.sendNotification(body.userId, body.type, body.channel, body.content);
  }

  @Get(':userId/logs')
  async getLogs(@Param('userId') userId: string) {
    return this.notificationsService.getNotificationLogs(userId);
  }

  @Get('stats')
  async getStats() {
    return this.notificationsService.getStats();
  }
}


 4.API Setup and MongoDB Connection

Now, integrate everything into the main `AppModule`.

typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { PreferencesModule } from './preferences/preferences.module';
import { NotificationsModule } from './notifications/notifications.module';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost:27017/notification-db'), // Use your MongoDB connection string
    PreferencesModule,
    NotificationsModule,
  ],
})
export class AppModule {}


