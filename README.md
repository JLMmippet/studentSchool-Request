# studentSchool-Request
Sample code for calling service to get student data

using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace Student.Data
{
    public abstract class StudentInquiryBase : StudentObjectBase
    {
        public StudentInquiryBase() { }

        //Method 1: Get student grade and test data from the district API 
        protected static StudentSchoolInfoResponse GetStudentSchoolInfo(ServiceLog sl, Array responseArray, string studentId, string teacherId, string userId, StudentInquiryDataEnum dataType)
        {
            List<Service.StudentKey> studentKeyList = new List<Service.StudentKey>();
            List<string> teacherKeyList = new List<string>();
            List<string> schoolKeyList = new List<string>();

            Tuple<List<Service.StudentKey>, List<string>, List<string>> keyLists;
            StudentSchoolInfoResponse response;

            //use response from StudentService to get student and school data from Service
            try
            {
                keyLists = CreateStudentSchoolKeyLists(studentKeyList, teacherKeyList, schoolKeyList, responseArray, studentId, teacherId, dataType);
 
                studentKeyList = keyLists.Item1;
                teacherKeyList = keyLists.Item2;
                schoolKeyList = keyLists.Item3;

                response = FetchStudentSchoolInfoAsync(sl, userId, studentKeyList, teacherKeyList, schoolKeyList).Result;
            }
            catch (Exception ex)
            {
                sl.WriteException(ex);
                throw;
            }
 
            return response;
        }
        
        //Method 2: Create Key Lists to use in requesting data from the service
        private static Tuple<List<Service.StudentKey>, List<string>, List<string>> CreateStudentSchoolKeyLists(List<Service.StudentKey> studentKeyList, List<string> teacherKeyList, List<string> schoolKeyList, Array responseArray, string studentId, string teacherId, StudentInquiryDataEnum dataType)
        {
            //if studentId was in UI request, we only need one studentId key in list
            if (!string.IsNullOrEmpty(studentId))
            {
                Service.StudentKey stuKey = new Service.StudentKey();
                stuKey.studentId= studentId.Substring(3, 5);
                stuKey.district = studentId.Substring(0, 3);
                stuKey.studentType = null;
                stuKey.zone = null;
 
                studentKeyList.Add(stuKey);
            }

            //if teacherId was in UI request, we only need one teacherId key in list
            if (!string.IsNullOrEmpty(teacherId))
            {
                teacherKeyList.Add(teacherId);
            }

            //create remaining request key lists, using initial StudentService response
            switch (dataType)
            {
                case StudentInquiryDataEnum.Public:
                    foreach (PublicStudentInquiry data in responseArray)
                    {
                        if (string.IsNullOrEmpty(studentId) && data.studentIdKey != null)
                        {
                            Service.StudentKey stuKey = new Service.StudentKey();
                            stuKey.studentId = data.studentIdKey.studentId;
                            stuKey.district = data.studentIdKey.district;
                            stuKey.studentType = null;
                            stuKey.zone = null;

                            //only add unique studentIds to list
                            if (!studentKeyList.Any(x => x.studentId == stuKey.studentId && x.district == stuKey.district))
                            {
                                studentKeyList.Add(stuKey);
                            }
                        }

                        if (string.IsNullOrEmpty(teacherId))
                        {
                            if (data.teacherId != null)
                            {
                                teacherKeyList.Add(data.teacherId.Length > 9 ? data.teacherId.Substring(0, 9) : data.teacherId);
                            }
                            else if (data.schoolId != null)
                            {
                                schoolKeyList.Add(data.schoolId);
                            }
                        }
                    }

                    //only add unique items to lists
                    teacherKeyList = teacherKeyList.Distinct().ToList();
                    schoolKeyList = schoolKeyList.Distinct().ToList();

                    break;
                case StudentInquiryDataEnum.Private:
                    foreach (PrivateInquiryDetails data in responseArray)
                    {
                        if (string.IsNullOrEmpty(studentId) && data.studentIdKey != null)
                        {
                            Service.StudentKey stuKey = new Service.StudentKey();
                            stuKey.studentId = data.studentIdKey.studentId;
                            stuKey.district = data.studentIdKey.district;
                            stuKey.studentIdType = null;
                            stuKey.zone = null;
                            
                            //only add unique studentIds to list
                            if (!studentKeyList.Any(x => x.studentId == stuKey.studentId && x.district == stuKey.district))
                            {
                                studentKeyList.Add(stuKey);
                            }
                        }

                        if (string.IsNullOrEmpty(teacherId))
                        {
                            if (data.teacherId != null)
                            {

                                teacherKeyList.Add(data.teacherId.Length > 9 ? data.teacherId.Substring(0, 9) : data.teacherId);
                            }
                            else if (data.schoolId != null)
                            {
                                schoolKeyList.Add(data.schoolId);
                            }
                        }
                    }

                    //only add unique items to lists
                    teacherKeyList = teacherKeyList.Distinct().ToList();
                    schoolKeyList = schoolKeyList.Distinct().ToList();

                    break;
            }

            return new Tuple<List<Service.StudentKey>, List<string>, List<string>>(studentKeyList, teacherKeyList, schoolKeyList);
        }

        //Method 3: Call the service
        private static async Task<StudentSchoolInfoResponse> FetchStudentSchoolInfoAsync(ServiceLog sl, string userId, List<Service.StudentKey> studentKeyList, List<string> teacherKeyList, List<string> schoolKeyList)
        {
            try
            {
                sl.WriteParameters($"StudentInquiryBase FetchStudentSchoolInfoAsyncParams: userId={userId}", $"studentKeyList={studentKeyList.Count}", $"teacherKeyList={teacherKeyList.Count}", $"schoolKeyList={schoolKeyList.Count}");

                //using key lists, fetch studentId and school details from Service
                using (ServiceClient Service = new ServiceClient())
                {
                    StudentSchoolInfoRequest Request = new StudentSchoolInfoRequest();

                    //if max not exceeded in any of the lists, for request, then only need one call to service
                    if (studentKeyList.Count() <= 10 && teacherKeyList.Count() <= 15 && schoolKeyList.Count() <= 15)
                    {
                        //create request                                          
                        StudentSchoolInfoReq requestList = GetBaseStudentSchoolInfoRequest(userId);
                        requestList.studentKeyList = studentKeyList.ToArray();
                        requestList.teacherKeyList = teacherKeyList.ToArray();
                        requestList.schoolIdList = schoolKeyList.ToArray();

                        request.studentSchoolInfoReq = requestList;

                        //execute the service call
                        DateTime isstart = DateTime.Now;
                        getStudentAndSchoolInformationResponse response = await Service.getStudentAndSchoolInformationAsync(request);
                        TimeSpan ists = DateTime.Now.Subtract(isstart);
                        sl.WriteCompletionMessage(Utilities.GetRequestIdFromResponseParams(response.StudentSchoolInfoResponse.studentSchoolInfoRes.commonResParam.ParamsList), ists, response.StudentSchoolInfoResponse.studentSchoolInfoRes != null ? 1 : 0);

                        return response.StudentSchoolInfoResponse;
                    }

                    //max exceeded for one+ of the request lists, so need multiple calls to the service

                    //create final response object
                    getStudentAndSchoolInformationResponse response2 = new getStudentAndSchoolInformationResponse();
                    StudentSchoolInfoResponse ssiResponse = new StudentSchoolInfoResponse();
                    response2.StudentSchoolInfoResponse = ssiResponse;
                    StudentSchoolInfoRes ssiRes = new StudentSchoolInfoRes();
                    response2.StudentSchoolInfoResponse.studentSchoolInfoRes = ssiRes;

                    //create temp response object
                    getStudentAndSchoolInformationResponse partialResponse;

                    StudentInfoDetails[] combinedStudentDetails = new StudentInfoDetails[studentKeyList.Count()];
                    int combinedSchoolListSize = teacherKeyList.Count() + schoolKeyList.Count();
                    SchoolInfoDetails[] combinedSchoolDetails = new SchoolInfoDetails[combinedSchoolListSize];

                    int studentListLeft = studentKeyList.Count();
                    int teacherListLeft = teacherKeyList.Count();
                    int schoolListLeft = schoolKeyList.Count();

                    int studentSkip = 0;
                    int teacherSkip = 0;
                    int schoolSkip = 0;

                    bool hasMoreStudents = studentSkip < studentKeyList.Count() ? true : false;
                    bool hasMoreTeachers = (teacherSkip + schoolSkip < combinedSchoolListSize) ? true : false;

                    //cycle thru student/teacher/school KeyLists and use them to call getStudentAndSchoolInformation, to get the additional field info

                    while (hasMoreStudents || hasMoreTeachers)
                    {
                        //clear request and create new one
                        request.studentSchoolInfoReq = null;
                        StudentSchoolInfoReq requestList = GetBaseStudentSchoolInfoRequest(userId);

                        if (hasMoreStudents)
                        {
                            //assign up to 10 students to requestList
                            requestList.studentKeyList = studentListLeft < 10 ? studentKeyList.Skip(studentSkip).Take(studentListLeft).ToArray() : studentKeyList.Skip(studentSkip).Take(10).ToArray();
                        }

                        if (hasMoreTeachers)
                        {
                            //assign up to 15 teacherIds or 15 schoolIds to requestList
                            if (teacherListLeft > 0) { requestList.teacherList = teacherListLeft < 15 ? teacherKeyList.Skip(teacherSkip).Take(teacherListLeft).ToArray() : teacherKeyList.Skip(teacherSkip).Take(15).ToArray(); }

                            if (schoolListLeft > 0) { requestList.schoolList = schoolListLeft < 15 ? schoolKeyList.Skip(schoolSkip).Take(schoolListLeft).ToArray() : schoolKeyList.Skip(schoolSkip).Take(15).ToArray(); }
                        }

                        request.studentSchoolInfoReq = requestList;

                        //execute the service call
                        partialResponse = null;
                        DateTime isstart = DateTime.Now;
                        partialResponse = await Service.getStudentAndSchoolInformationAsync(request);
                        TimeSpan ists = DateTime.Now.Subtract(isstart);
                        sl.WriteCompletionMessage(Utilities.GetRequestIdFromResponseParams(partialResponse.StudentSchoolInfoResponse.studentSchoolInfoRes.commonResParam.ParamsList), ists, partialResponse.StudentSchoolInfoResponse.studentSchoolInfoRes != null ? 1 : 0);

                        if (hasMoreStudents)
                        {
                            //copy response to combinedStudentDetails array
                            partialResponse.StudentSchoolInfoResponse.studentSchoolInfoRes.StudentInfoDetailsList.CopyTo(combinedStudentDetails, studentSkip);

                            studentSkip = studentListLeft < 10 ? studentSkip += studentListLeft : studentSkip += 10;
                            studentListLeft = studentListLeft < 10 ? 0 : studentListLeft -= 10;
                        }

                        if (hasMoreTeachers)
                        {
                           //copy response to combinedSchoolDetails array 
                                    partialResponse.StudentSchoolInfoResponse.studentSchoolInfoRes.SchoolInfoDetailsList.CopyTo(combinedSchoolDetails, teacherSkip + schoolSkip);

                            teacherSkip = teacherListLeft < 15 ? teacherSkip += teacherListLeft : teacherSkip += 15;
                            teacherListLeft = teacherListLeft < 15 ? 0 : teacherListLeft -= 15;

                            schoolSkip = schoolListLeft < 15 ? schoolSkip += schoolListLeft : schoolSkip += 15;
                            schoolListLeft = schoolListLeft < 15 ? 0 : schoolListLeft -= 15;
                        }

                        hasMoreStudents = studentSkip < studentKeyList.Count() ? true : false;
                        hasMoreTeachers = (teacherSkip + schoolSkip < combinedSchoolListSize) ? true : false;
                    }

                    //assign all student and school details to final response
                    response2.StudentSchoolInfoResponse.studentSchoolInfoRes.StudentInfoDetailsList = combinedStudentDetails;
                    response2.StudentSchoolInfoResponse.studentSchoolInfoRes.SchoolInfoDetailsList = combinedSchoolDetails;

                    return response2.StudentSchoolInfoResponse;
                }
            }
            catch (AggregateException aex)      //needed? use when there are more than 2 async tasks returned to this method
            {
                sl.WriteException(aex);
                throw aex.GetRootException();
            }
            catch (Exception ex)
            {
                sl.WriteException(ex);
                throw;
            }
        }
    }
}
