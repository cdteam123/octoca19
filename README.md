using System;
using System.Data;
using System.Diagnostics;
using System.IO;
using System.Net;
using Ayehu.Sdk.ActivityCreation.Extension;
using Ayehu.Sdk.ActivityCreation.Interfaces;

namespace Ayehu.Sdk.ActivityCreation
{
    public class GetProcess : IActivity
    {
        public string target = "";

        public ICustomActivityResult Execute()
        {
            string powerShellPath = @"C:\Program Files\PowerShell\7\pwsh.exe";
            string command = "-Command \"Get-WmiObject -Class Win32_Processor | Format-Table Caption, DeviceID, Manufacturer, MaxClockSpeed, Name, SocketDesignation\"";

            ProcessStartInfo processStartInfo = new ProcessStartInfo
            {
                FileName = powerShellPath,
                Arguments = command,
                RedirectStandardOutput = true,
                RedirectStandardError = true,
                UseShellExecute = false,
                CreateNoWindow = true
            };

            try
            {
                using (Process process = new Process())
                {
                    process.StartInfo = processStartInfo;
                    process.Start();

                    string output = process.StandardOutput.ReadToEnd();
                    string error = process.StandardError.ReadToEnd();

                    process.WaitForExit();

                    if (!string.IsNullOrEmpty(error))
                    {
                        return this.GenerateActivityResult("Error occurred: " + error);
                    }
                    else
                    {
                        output = RemoveEscapeSequences(output);

                        // Convert output to DataTable
                        GetProcessHelper helper = new GetProcessHelper();
                        DataTable dataTable = helper.ConvertOutputToDataTable(output);

                        return this.GenerateActivityResult(dataTable);
                    }
                }
            }
            catch (Exception ex)
            {
                return this.GenerateActivityResult("An error occurred: " + ex.Message);
            }
        }

        private string RemoveEscapeSequences(string input)
        {
            input = input.Replace("\u001b[32;1m", string.Empty);
            input = input.Replace("\u001b[0m", string.Empty);
            input = input.Replace("\u001b", string.Empty);
            return input;
        }
    }

    public class GetProcessHelper
    {
        public DataTable ConvertOutputToDataTable(string output)
        {
            DataTable dataTable = new DataTable();
            string[] lines = output.Split(new[] { "\r\n", "\r", "\n" }, StringSplitOptions.RemoveEmptyEntries);

            if (lines.Length < 2)
            {
                return dataTable;
            }

            string[] headers = lines[0].Split(new[] { " " }, StringSplitOptions.RemoveEmptyEntries);
            foreach (string header in headers)
            {
                dataTable.Columns.Add(header.Trim());
            }

            for (int i = 1; i < lines.Length; i++)
            {
                string[] values = lines[i].Split(new[] { " " }, StringSplitOptions.RemoveEmptyEntries);
                if (values.Length == headers.Length)
                {
                    DataRow dataRow = dataTable.NewRow();
                    for (int j = 0; j < headers.Length; j++)
                    {
                        dataRow[j] = values[j].Trim();
                    }
                    dataTable.Rows.Add(dataRow);
                }
            }

            return dataTable;
        }
    }
}
